---
name: image-preprocessing-and-tiling
description: Build and debug image preprocessing pipelines for deep learning when images are large, multi-channel, or scientifically calibrated - satellite/aerial GeoTIFF, Sentinel/Landsat multispectral stacks, whole-slide pathology (SVS/NDPI/OpenSlide), CT/MRI volumes, or any raster too big for the GPU. Use when the task mentions tiling, patching, sliding window, chip extraction, stride/overlap, seams when stitching predictions, reassembling tiles into a full-scene mask, normalization statistics, per-channel mean/std, percentile clipping, nodata poisoning stats, albumentations, image+mask augmentation, additional_targets, class imbalance in segmentation, empty/background tiles, slow DataLoader num_workers on Windows, worker seeding and reproducible augmentation, or converting a dataset to memmap numpy / WebDataset / LMDB / zarr. Also use when a segmentation or dense-prediction model trains fine but produces grid-shaped seams, all-background outputs, or scores far worse at inference than in validation.
---

# Image Preprocessing and Tiling

Failure modes here are silent, not exceptions: a `-9999` nodata sentinel shifts a channel mean by
orders of magnitude, a mask gets bilinear-interpolated into non-integer class ids, augmentation
randomness is duplicated across eight workers, inference normalization differs from training by a
factor of 255. The model still trains. It just does not work.

---

## 1. Tiling: choosing tile size and stride

| Decision | Rule |
|---|---|
| Tile size | Largest power-of-two that fits batch size >= 8 in VRAM. 512 for most segmentation UNets at bs 8-16 on 24GB; 256 if you need bs 32+; 1024 only if context genuinely matters (e.g. large fields, tumor architecture). |
| Training stride | Irrelevant if you sample random crops. If you pre-cut a fixed tile grid, use stride == tile (no overlap) and rely on random offsets per epoch instead. |
| Inference stride | `tile // 2` is the default worth defending. It is the minimum overlap at which a Hann window sums to a constant (COLA), and it doubles compute in each axis (4x total). `tile * 3 // 4` (25% overlap) is the cheap compromise. |
| Receptive field check | Overlap should be >= the model's effective receptive field on each side, otherwise pixels near a tile edge are predicted from truncated context. For a UNet with 4 downsamples, that is roughly 100-200 px. Overlap of 128 px at tile 512 satisfies this; overlap of 16 px does not. |

**Why overlap at inference:** a pixel 3 px from a tile border saw almost no context on one side, so
its prediction is systematically worse. With stride == tile you keep exactly those bad predictions
along every seam - that is the visible grid pattern. Overlap lets you replace edge predictions with
center predictions from the neighbour. Naive overwrite is not enough: writing one tile over another
just moves the discontinuity to the write boundary. You need a smooth weight.

---

## 2. Reassembly with feathered (Hann) blending

```python
import numpy as np

def hann2d(th, tw, eps=1e-6):
    # np.hanning(N) is EXACTLY 0 at both endpoints. If a border pixel is covered
    # by only one tile, its accumulated weight would be 0 -> 0/0 -> NaN.
    # Sampling the interior of a length-(N+2) window avoids the zeros entirely.
    wy = np.hanning(th + 2)[1:-1]
    wx = np.hanning(tw + 2)[1:-1]
    return np.maximum(np.outer(wy, wx), eps).astype(np.float32)

def tile_starts(size, tile, stride):
    """Start indices covering [0, size). Final tile is SHIFTED to end exactly at
    `size` instead of padding - so no tile ever contains synthetic pixels."""
    if size <= tile:
        return [0]
    starts = list(range(0, size - tile + 1, stride))
    if starts[-1] != size - tile:
        starts.append(size - tile)
    return starts

def predict_scene(image, model_fn, tile=512, stride=256, n_classes=1):
    """image: (C, H, W) float32, already normalized. model_fn: (C,t,t)->(K,t,t)."""
    C, H, W = image.shape
    if H < tile or W < tile:                         # scene smaller than one tile:
        pad = ((0, 0), (0, max(0, tile - H)), (0, max(0, tile - W)))
        big = np.pad(image, pad, mode="symmetric")   # pad up, predict, crop back
        return predict_scene(big, model_fn, tile, stride, n_classes)[:, :H, :W]
    acc  = np.zeros((n_classes, H, W), np.float32)   # weighted sum of logits/probs
    wacc = np.zeros((1, H, W), np.float32)           # accumulated weight
    win  = hann2d(tile, tile)[None]                  # (1,t,t) broadcasts over K

    for y in tile_starts(H, tile, stride):
        for x in tile_starts(W, tile, stride):
            patch = image[:, y:y+tile, x:x+tile]
            pred  = model_fn(patch)                  # (K, t, t)
            acc[:,  y:y+tile, x:x+tile] += pred * win
            wacc[:, y:y+tile, x:x+tile] += win
    return acc / wacc
```

Non-obvious points in that code:

- **Divide by accumulated weight, not by a count.** The shifted final tile overlaps its neighbour by
  an irregular amount; some pixels get 2 tiles, some 4, some 3. Only `acc / wacc` is correct for all
  of them, and it makes the irregular last-row/last-column stride a non-issue.
- **Blend probabilities or logits, not argmax labels.** Averaging class indices is meaningless.
  Prefer averaging softmax probabilities (bounded, well-behaved); averaging logits is also fine and
  slightly sharper, but do not mix the two across tiles.
- **`acc` must be float32 minimum.** float16 accumulation over 4 overlapping tiles of a 20k x 20k
  scene loses precision visibly in low-probability regions.
- **Memory.** `acc` for a 20000x20000 scene with 10 classes in float32 is 16 GB. Use
  `np.lib.format.open_memmap`, or process the scene in row-blocks.
- The torch equivalent is `F.unfold`/`F.fold`: `F.fold` sums overlaps, so fold `pred * win` and
  separately fold `win` broadcast to the same shape, then divide. But `unfold` materializes every
  tile at once and will OOM on a large scene.
- **3D volumes:** do not hand-roll this. `monai.inferers.sliding_window_inference(inputs, roi_size,
  sw_batch_size, predictor, overlap=0.5, mode="gaussian")` is the same algorithm with a Gaussian
  instead of a Hann window, batched patches, and `device=`/`sw_device=` so the accumulator can live
  on CPU while the model runs on GPU. `mode="constant"` is the seam-producing default trap.
- **Round-trip assertion** (section 10): with `model_fn = lambda p: p`, the reassembled output must
  equal the input to ~1e-5. If it does not, your seams are a tiling bug, not a model bug.

---

## 3. Edges and remainders

Two valid strategies for `H % tile != 0`:

| Strategy | Use when | Cost |
|---|---|---|
| **Shifted final tile** (`tile_starts` above) | Inference, always, when `H >= tile` | Extra overlap in the last row/col; free because weighted blending handles it. |
| **Pad then crop** | `H < tile`; or fully-convolutional models needing a fixed grid; or training where you want the true border seen | Introduces synthetic pixels the model must learn to ignore. |

If you pad, **pad by reflection, not zeros**:

```python
pad_h, pad_w = (-H) % tile, (-W) % tile
padded = np.pad(image, ((0,0), (0,pad_h), (0,pad_w)), mode="reflect")
# then crop back: out = out[:, :H, :W]
```

- Zero padding creates a hard step from real radiance to exactly 0. Convolutions near the border see
  an edge that exists in *every* padded sample at the same place, so the network learns a
  border-specific response - a bright or dark 5-20 px rim on predictions and worse metrics near
  scene edges. 0 is also a legitimate reflectance value and a common nodata sentinel.
- `np.pad(mode="reflect")` tolerates `pad_width >= dim` (it reflects repeatedly), but the same
  request raises in `torch.nn.functional.pad(mode="reflect")` and in `cv2.copyMakeBorder`, both of
  which require `pad < dim` (`pad < dim - 1` for `BORDER_REFLECT_101`). Padding a 3-px strip by 100
  therefore works offline in numpy and blows up inside the model or an albumentations transform.
  In numpy, `mode="symmetric"` repeats the edge sample under no length constraint - the robust
  default. Either way, padding a strip by more than its own width fabricates most of the tile; drop
  such slivers instead of padding them.
- In OpenCV/albumentations the equivalent is `cv2.BORDER_REFLECT_101` (= numpy `reflect`) vs
  `cv2.BORDER_REFLECT` (= numpy `symmetric`). `BORDER_CONSTANT` is the zero-padding trap.
- **Pad the mask with an ignore label, not with class 0.** Reflect-padding the image but zero-padding
  the mask teaches the model that reflected content is background. Use `fill_mask=255` (or your
  ignore_index) and set `ignore_index=255` in the loss.

---

## 4. Normalization

### The four rules

1. **Statistics come from the training split only.** Train+val+test is leakage - small, but exactly
   what reviewers ask about, and not small at all with a distribution-shifted test set.
2. **Per channel.** A 12-band Sentinel-2 stack has band means spanning an order of magnitude; one
   scalar mean/std flattens the informative bands into noise.
3. **ImageNet statistics are only valid for 8-bit RGB natural photographs.** For multispectral, SAR
   (dB, can be negative), thermal, DEM (metres), or 16-bit medical data they are meaningless. Even
   for 3-band RGB *satellite* imagery they are wrong - the radiometry is different.
4. **Save the statistics next to the checkpoint and reload them at inference.** Recomputing them
   from the inference scene is the most common source of train/inference skew, and it presents
   exactly as a model that "doesn't generalize".

### Computing stats without poisoning them

```python
import json, numpy as np, rasterio

C = 12
n = np.zeros(C, np.float64); s = np.zeros(C, np.float64); ss = np.zeros(C, np.float64)
SHIFT = None  # provisional mean, prevents catastrophic cancellation for large values

for path in TRAIN_PATHS:                        # TRAIN SPLIT ONLY
    with rasterio.open(path) as src:
        a = src.read(masked=True)               # masked=True honours the nodata tag
    a = a.astype(np.float64)
    valid = ~np.ma.getmaskarray(a)
    # Also drop known sentinels the file failed to declare:
    valid &= np.isfinite(a) & (a != -9999) & (a != 0)   # tune per dataset!
    if SHIFT is None:
        SHIFT = np.array([a[c][valid[c]].mean() if valid[c].any() else 0.0
                          for c in range(C)])
    d = np.where(valid, a - SHIFT[:, None, None], 0.0)
    n  += valid.sum(axis=(1, 2), dtype=np.float64)
    s  += d.sum(axis=(1, 2), dtype=np.float64)
    ss += (d * d).sum(axis=(1, 2), dtype=np.float64)

assert (n > 0).all(), n          # a channel with 0 valid pixels -> NaN stats, silently
mean = SHIFT + s / n
std  = np.sqrt(np.maximum(ss / n - (s / n) ** 2, 0.0))
json.dump({"mean": mean.tolist(), "std": std.tolist(), "n": n.tolist()},
          open("norm_stats.json", "w"))
```

Gotchas encoded above:

- **`src.read()` returns nodata pixels as their raw sentinel value.** `masked=True` gives a
  `MaskedArray` using the file's *declared* nodata - but many GeoTIFFs declare nothing, or declare 0
  while also using `-9999` in some tiles. Print `src.nodata`, then histogram one scene and look for
  a spike at 0 / -9999 / 65535 before trusting it. A `-9999` sentinel in **1%** of pixels drags a
  band with mean 1500 / std 300 to mean 1385 and std 1182 - the mean moves 8%, the std nearly **4x**.
  Every downstream z-score is then compressed toward zero.
- **Always pass `dtype=np.float64` to `.sum()`.** Integer reductions accumulate in the platform's
  default integer, which is 32-bit on Windows under numpy < 2.0 - `uint16` sums overflow silently
  past 4.3e9 (a single 20k x 20k band of mid-range reflectance). NumPy >= 2.0 made the Windows
  default 64-bit, so this bug appears and disappears with the numpy version; pin the accumulator.
- **`ss/n - mean^2` is numerically fragile** for uint16 reflectance (values ~1e4, N ~1e10). The
  provisional-mean shift above costs nothing and removes the problem; `np.maximum(..., 0)` guards
  the residual negative variance.
- Do **not** compute stats over the tiles your foreground-biased sampler emits - that biases the
  statistics toward the rare class. Compute over the full training scenes.

### Percentile clipping

For imagery with heavy tails (specular water glint, clouds, sensor spikes, saturated pathology
whites), z-scoring alone leaves 99.9% of pixels squeezed into a narrow band.

```python
# Sample ~1e7 valid pixels per channel from the TRAIN split, then:
p1, p99 = np.percentile(sampled, [1, 99], axis=0)      # shape (C,)
x = np.clip(x, p1[:, None, None], p99[:, None, None])
x = (x - p1[:, None, None]) / (p99 - p1)[:, None, None]   # -> [0, 1]
```

Order matters: **clip first, then scale**. Compute the percentiles globally over the training set,
never per-scene at inference - per-scene percentiles make the same physical radiance map to
different network inputs depending on what else is in the scene, destroying cross-scene
comparability. (Per-scene is acceptable only for visualization, or when sensor gain genuinely varies
per acquisition and you have decided to normalize it away.) The right percentile is data dependent:
1/99 is fine on cloud-free agricultural scenes; 2/98 is often needed with water glint or cloud.

### Domain notes

- **CT** is calibrated in Hounsfield units. Window first (`soft tissue` ~ center 40 / width 400;
  `lung` ~ -600 / 1500; `bone` ~ 400 / 1800), then scale to [0,1]. Global stats are meaningful.
- **MRI** intensities are in arbitrary units that vary by scanner, coil and sequence. Global
  mean/std across a dataset is wrong. Normalize **per volume** (z-score over the brain mask, or
  scale by the 99th percentile inside the mask), and do it after skull-stripping / bias-field
  correction if you are doing those.
- **SAR** in linear power is log-normal - convert to dB (`10*log10`) before z-scoring, and handle
  zeros before the log.
- **Voxel spacing before patching (CT/MRI).** A 96x96x96 patch is a different physical volume in
  every scan until you resample to a fixed spacing (e.g. 1x1x1 mm, or the dataset median as nnU-Net
  does). CT series are routinely 0.7x0.7x5 mm - patching that anisotropic grid directly means the
  z-axis receptive field covers 10x the anatomy the in-plane one does. Resample the image with
  linear/spline interpolation and the label map with **nearest neighbour**, and re-check the label
  set afterwards. Also apply the DICOM `RescaleSlope`/`RescaleIntercept` before treating values as
  HU; `pydicom` does not apply them for you (`apply_modality_lut` does).
- **Adapting a pretrained RGB encoder to N bands.** Do not feed 12 bands through an ImageNet-stat
  normalization. Compute your own N-channel stats, then inflate the stem weights:
  `w_new = w.mean(1, keepdim=True).repeat(1, N, 1, 1) * (3 / N)` - the `3/N` keeps the stem's output
  magnitude where the pretrained downstream layers expect it. Without the rescale, a 12-band stem
  emits ~4x the activation scale and the first epochs are spent undoing it.

---

## 5. Augmentation with albumentations

```python
import cv2, numpy as np, albumentations as A
from albumentations.pytorch import ToTensorV2

MEAN, STD = stats["mean"], stats["std"]   # loaded from norm_stats.json

train_tf = A.Compose([
    # --- geometric: applied identically to image AND mask ---
    A.PadIfNeeded(min_height=512, min_width=512,
                  border_mode=cv2.BORDER_REFLECT_101),
    A.RandomCrop(height=512, width=512),
    A.HorizontalFlip(p=0.5),
    A.VerticalFlip(p=0.5),        # overhead imagery ONLY - see section 6
    A.RandomRotate90(p=0.5),      # overhead imagery ONLY
    A.Affine(scale=(0.9, 1.1), translate_percent=(-0.05, 0.05),
             rotate=(-15, 15), p=0.5),
    # --- photometric: image only, albumentations handles this automatically ---
    A.RandomBrightnessContrast(brightness_limit=0.2, contrast_limit=0.2, p=0.3),
    # --- normalization + tensor ---
    A.Normalize(mean=MEAN, std=STD, max_pixel_value=1.0),
    ToTensorV2(),
])

out = train_tf(image=img_hwc, mask=mask_hw)   # img (H,W,C), mask (H,W) int
x, y = out["image"], out["mask"]               # (C,H,W) float32, (H,W) int64-able
```

The traps, in order of how often they bite:

- **`A.Normalize(max_pixel_value=255.0)` is the default.** If your array is already float in [0,1],
  or is 16-bit reflectance you scaled yourself, the default divides by 255 *again* and your inputs
  land near 0.004. The model trains, slowly, to a mediocre score. Pass `max_pixel_value=1.0`
  whenever the input is not raw uint8.
- **`ToTensorV2` does NOT divide by 255** (unlike `torchvision.transforms.ToTensor`); it only
  transposes HWC->CHW. Porting a torchvision pipeline without adding an explicit scale is a silent
  255x error.
- **`ToTensorV2` does not transpose the mask** unless you pass `transpose_mask=True`. A `(H,W,K)`
  one-hot mask stays `(H,W,K)` and your loss silently broadcasts wrong.
- **Masks are interpolated with nearest neighbour** by albumentations' geometric transforms, so
  class ids stay integral - but only if you pass the array as `mask=`. If you sneak the mask through
  as a 4th image channel to "save a call", it gets bilinear-interpolated and you get class id 3.7.
  Assert `set(np.unique(y)) <= set(range(K)) | {ignore_index}` after augmentation.
- **`additional_targets`** is how you get a second mask, a second image, or a change-detection pair
  transformed with the *same* random parameters:

  ```python
  tf = A.Compose([...], additional_targets={"image_t2": "image", "mask_prev": "mask"})
  out = tf(image=a, image_t2=b, mask=m, mask_prev=mp)
  ```

  Without it, calling the transform twice draws two independent random states and image and mask are
  no longer aligned - which presents as a model that cannot learn, not as a bug.
- **Photometric augs must not touch the mask.** Albumentations enforces this for its built-ins
  (`apply_to_mask` is identity for brightness/contrast/blur/noise). For a custom transform, subclass
  `A.ImageOnlyTransform`, not `A.DualTransform`.
- **Version rename (albumentations 2.0):** the border/fill kwargs were inconsistent before 2.0 and
  were unified in it. Pre-2.0: most geometric transforms (`PadIfNeeded`, `Rotate`, `ShiftScaleRotate`)
  took `border_mode=` / `value=` / `mask_value=`, while `A.Affine` took `mode=` / `cval=` /
  `cval_mask=`. From 2.0 everything is `border_mode=` / `fill=` / `fill_mask=`. Print `A.__version__`
  and `inspect.signature(A.Affine.__init__)` rather than guessing; an unknown kwarg is silently
  ignored in some versions and raises in others, so a wrong name can mean you were zero-padding all
  along.
- **>3 channels:** geometric transforms and `Normalize` handle arbitrary channel counts. Colour-space
  transforms (`HueSaturationValue`, `RGBShift`, `CLAHE`, `ToGray`, most `RandomFog`-style ones)
  assume 3 or 1 channels and will raise or silently mangle a 12-band stack. Verify on one sample.
  Leave `is_check_shapes=True` (default) on - it catches image/mask shape mismatch.

---

## 6. Augmentations that are wrong for your domain

| Augmentation | Wrong when | Why |
|---|---|---|
| `HorizontalFlip` | Text, digits, characters, any chirality-bearing content; medical images where left/right is diagnostic (situs inversus, side-specific lesions, laterality labels) | Mirrored "3" is not a 3. A flipped chest X-ray moves the heart to the right side - the model learns that heart-on-right is normal. |
| `VerticalFlip` / `RandomRotate90` | Ground-level natural photos, portraits, documents, most microscopy with a defined stage orientation | Gravity gives natural scenes a canonical up. Upside-down cars are off-manifold and waste capacity. |
| `VerticalFlip` / `RandomRotate90` | **Fine and recommended** for overhead/satellite/aerial imagery, whole-slide pathology, and most 2D microscopy | There is no privileged orientation - the full dihedral group D4 is a free 8x data multiplier. |
| Arbitrary `rotate` | Medical volumes with laterality or standardized acquisition planes; anything where you later report an anatomical measurement | Rotation past a few degrees breaks the plane convention the labels assume. Keep to +/-10-15 deg. |
| `ColorJitter` / `RandomBrightnessContrast` / `HueSaturationValue` | Multispectral, SAR, thermal, CT (HU), any calibrated intensity | The pixel value *is* the physical measurement. Jittering it teaches invariance to the exact signal you are trying to measure - NDVI-like band ratios are destroyed. |
| `ColorJitter` | **Fine and important** for H&E pathology (stain variation) and RGB drone/street imagery | Stain and illumination genuinely vary between scanners/sites. Prefer stain-specific augmentation (HED colour-space jitter) over naive RGB jitter for pathology. |
| `Normalize` before geometric transforms | Always | Rotation/scale interpolate; doing it on normalized data is fine numerically but any constant-fill border becomes a "0 = mean" pixel rather than a flaggable value. Keep `Normalize` + `ToTensorV2` last. |
| `RandomResizedCrop` with aggressive scale | Remote sensing with a fixed GSD; anything where object size is a physical quantity | If 1 px = 10 m always, scale jitter breaks the relationship between object size and class. Mild scale (0.9-1.1) only. |

---

## 7. Class imbalance in dense prediction

Uniformly sampling tile origins from a scene where the target covers 0.5% of pixels gives you tiles
that are ~85-95% completely empty. The gradient is dominated by background, the loss drops fast, and
the model converges to predicting all-background - which scores 99.5% pixel accuracy and 0 IoU.

**Precompute a tile index with foreground fraction, then sample it.**

```python
# Offline, once - tile_starts() is from section 2:
index = []
for scene_id, mask in enumerate(train_masks):            # each (H, W) uint8
    H, W = mask.shape
    for y in tile_starts(H, TILE, TILE):
        for x in tile_starts(W, TILE, TILE):
            fg = float((mask[y:y+TILE, x:x+TILE] > 0).mean())
            index.append({"scene": scene_id, "y": y, "x": x, "fg": fg})

# In the Dataset: 50/50 mix
fg_pool = [i for i, r in enumerate(index) if r["fg"] > 0.01]
bg_pool = [i for i, r in enumerate(index) if r["fg"] <= 0.01]

def __getitem__(self, i):
    pool = fg_pool if (i % 2 == 0) else bg_pool
    rec = index[random.choice(pool)]   # global RNG: PyTorch reseeds it per worker.
    ...                                # An RNG built in __init__ would not be - section 9.
```

`A.CropNonEmptyMaskIfExists(height, width, p=1.0)` crops around a non-zero mask region when one
exists and falls back to a random crop otherwise - a fast baseline, but it centres crops on
foreground, biasing object position toward the tile centre. Mix it at `p=0.5` with plain
`RandomCrop` to avoid that.

- **Oversampling foreground changes the class prior**, so predicted probabilities are miscalibrated
  relative to the real scene. Tune any decision threshold on a validation set sampled the way
  deployment samples (full scenes), not on the oversampled tiles.
- Do not stack mechanisms: aggressive oversampling **plus** class-weighted loss **plus** focal loss
  overshoots into over-prediction. Pick one primary (usually sampling) and one mild secondary (Dice).
- Keep genuinely-empty tiles in the mix. A model that never sees pure background produces false
  positives everywhere on the real scene.

---

## 8. DataLoader performance

Decoding is almost always the bottleneck - not the GPU, not the disk. A 512x512 JPEG decode is
~2-5 ms; a compressed-tile GeoTIFF window read 10-50 ms; an OpenSlide `read_region` at level 0
50-200 ms. 8 workers x 3 ms is ~2600 img/s (more than most models consume); 8 workers x 50 ms is
160 img/s and your GPU sits idle.

```python
loader = DataLoader(
    ds, batch_size=16, shuffle=True,
    num_workers=8,              # start at physical cores; measure, do not guess
    pin_memory=True,            # only helps if you then use .to(dev, non_blocking=True)
    persistent_workers=True,    # essential on Windows: avoids respawning every epoch
    prefetch_factor=4,          # batches queued per worker; raise if timings are spiky
    drop_last=True,
)
```

### Storage format decision table

| Format | Wins when | Cost |
|---|---|---|
| Loose JPEG/PNG | Small datasets, RGB, prototyping | Slow on network/spinning storage; millions of small files kill NTFS |
| **`np.memmap` / `np.lib.format.open_memmap`** | Fixed-size tiles, uncompressed, fits on local disk. Fastest possible random access; OS page cache does the work | No compression - a 100k x (12,512,512) uint16 store is huge |
| **WebDataset (tar shards)** | Large datasets, network/cloud storage, multi-node. Sequential reads, near-linear scaling | Only *approximate* shuffling (shuffle buffer + shard shuffle); awkward for foreground-biased sampling |
| **LMDB** | Millions of small samples, need true random access with compression | Single-writer; DB file size must be preallocated (`map_size`); one more dependency |
| **zarr** (+ Dask) | Huge N-dimensional arrays (time series of scenes, 3D volumes) where you slice arbitrary windows; chunked + compressed | Chunk shape must match your access pattern or you decompress 10x more than you read |
| Native GeoTIFF via rasterio windows | You cannot afford to duplicate a multi-TB archive | Ensure the file is *tiled* (not striped) and internally overviewed, else a 512x512 window read decodes whole 20000-px strips |

Rule of thumb: convert to memmap or WebDataset when a profiling run shows GPU utilization below
~70% and worker count is already at core count.

### Whole-slide images (OpenSlide)

- **Windows install.** Python 3.8+ ignores `PATH` when resolving extension DLLs, so `import
  openslide` fails with a DLL load error until you point at the unpacked binaries first:

  ```python
  import os
  with os.add_dll_directory(r"C:\openslide\bin"):   # before the import, every process
      import openslide
  ```

  Because workers spawn, this must run at module import, not once in `main()`.
- **`read_region((x, y), level, (w, h))` takes `(x, y)` in the level-0 frame** no matter which
  `level` you pass, while `(w, h)` is in that level's pixels. Scaling the origin by
  `slide.level_downsamples[level]` "to be consistent" is the classic bug: you read a region
  `downsample^2` away from where you meant, and every tile is misaligned with its annotation.
- **It returns RGBA.** `.convert("RGB")` explicitly - the alpha channel is 0 outside the scanned
  area and naive `np.array(region)[..., :3]` leaves those regions black, which your tissue filter
  then happily accepts as "dark = tissue".
- **Filter glass before tiling.** 70-90% of a slide is background. Otsu-threshold the saturation
  channel of a low-resolution level (`slide.get_thumbnail(...)` or `level_count - 1`), then keep only
  tiles whose tissue fraction exceeds ~0.1. Doing this offline into a tile index (section 7) is the
  difference between a 3-hour and a 30-hour epoch.
- **Normalize magnification, not pixels.** `slide.properties[openslide.PROPERTY_NAME_MPP_X]` is
  0.25 um/px at 40x and 0.5 at 20x, and it is missing on some NDPI/scanner exports. A fixed 512-px
  tile therefore covers 2x different tissue across a multi-site cohort - resample to a target MPP
  and fail loudly when the property is absent rather than assuming 40x.

### Windows specifics

- Windows uses `spawn`, not `fork`. **`num_workers > 0` requires your training entry point to be
  guarded**, or each worker re-executes the script and you get an infinite spawn storm (usually
  presenting as a `RuntimeError` about the current process finishing bootstrapping, or as the
  machine simply freezing):

  ```python
  if __name__ == "__main__":
      main()
  ```
- Everything crossing the process boundary must be **picklable**: no lambdas in `worker_init_fn`,
  no local closures in `collate_fn`, no `partial` over a nested function. Module-level functions only.
- Spawn re-imports your module in every worker, so module-level heavy work (loading a big index,
  importing torch, opening a DB) is paid `num_workers` times per spawn. Worker startup is ~1-3 s
  each on Windows vs ~50 ms on Linux, so `num_workers=16` can cost 30 s per epoch in pure startup;
  4-8 is the sweet spot even on a 16-core machine, and `persistent_workers=True` (pay once per run,
  not once per epoch) is often the single biggest wall-clock win.
- **Open file handles lazily, inside the worker.** `h5py.File`, `rasterio.open`, `lmdb.open` and
  OpenSlide handles created in `Dataset.__init__` are either unpicklable (spawn crashes with a
  confusing pickling error) or shared unsafely. Standard pattern:

  ```python
  def __getitem__(self, i):
      if self._h5 is None:              # set to None in __init__
          self._h5 = h5py.File(self.path, "r")
      ...
  ```
- Add `cv2.setNumThreads(0)` at module level (and `OMP_NUM_THREADS=1`). OpenCV's thread pool times
  8 worker processes oversubscribes the CPU and can make the loader *slower* than `num_workers=0`.

---

## 9. Determinism in the data path

```python
import random, numpy as np, torch

def seed_worker(worker_id):                  # must be module-level (Windows pickling)
    s = torch.initial_seed() % 2**32         # per-worker, derived from base_seed
    np.random.seed(s)
    random.seed(s)
    tf = torch.utils.data.get_worker_info().dataset.transform
    if hasattr(tf, "set_random_seed"):       # albumentations >= 2.0, see below
        tf.set_random_seed(s)

g = torch.Generator()
g.manual_seed(1337)                          # controls the shuffle order
loader = DataLoader(ds, ..., worker_init_fn=seed_worker, generator=g)
```

- The `generator=` argument seeds the **sampler** (which indices, in what order). `worker_init_fn`
  seeds the **augmentation** RNGs. You need both; they are independent.
- Modern PyTorch does seed `random` and numpy's global RNG per worker, but a `np.random.RandomState`
  or `random.Random` instance you construct in `Dataset.__init__` is copied to every worker
  identically - so all 8 workers draw the *same* augmentation sequence. Construct per-worker RNGs
  lazily inside `__getitem__`/`worker_init_fn`, or use the global RNG.
- **Albumentations changed RNG model in 2.0 and `seed_worker` no longer covers it.** Pre-2.0
  transforms drew from the global `random` / `numpy` RNGs, so seeding the globals per worker was
  enough. From 2.0 (and late 1.4.x) every transform owns a per-instance RNG fixed at *construction*
  time, plus `A.Compose(..., seed=N)`. A `Compose` built in the parent process and shipped to
  workers therefore carries the **same** RNG state into all of them - eight workers, one
  augmentation sequence - and nothing in `worker_init_fn` that touches globals will change that.
  Either reseed the pipeline per worker (the `set_random_seed` lines above) or construct the
  `Compose` lazily inside the worker.
  Symptom if you miss it: the k-th sample produced by *every* worker gets identical augmentation
  parameters, so each batch contains `num_workers` copies of the same flip/rotate/brightness draw
  applied to different tiles. Verify by transforming one fixed array in each worker and comparing
  hashes - they must differ.
- To reproduce one exact sample, use `A.ReplayCompose` and store the returned `replay` dict.
- Reproducing a run needs the sampler seed, the worker seed scheme, `num_workers`, the dataset
  ordering, and library versions - log all of them. Changing `num_workers` changes which sample gets
  which seed, so a run is not reproducible across a worker-count change even with identical seeds.
- `torch.use_deterministic_algorithms(True)` plus `CUBLAS_WORKSPACE_CONFIG=:4096:8` handles the model
  side; it does nothing for the data path.

---

## 10. Debugging: how to actually verify the pipeline

Do these before training anything. Each one has caught a real, silent, model-killing bug.

**1. Dump a batch and look at it with your eyes.**

```python
import torchvision
x, y = next(iter(train_loader))
vis = x[:, :3]                                     # pick RGB bands for multispectral
vis = (vis - vis.amin()) / (vis.amax() - vis.amin() + 1e-8)
torchvision.utils.save_image(torchvision.utils.make_grid(vis, nrow=4), "batch.png")
torchvision.utils.save_image(
    torchvision.utils.make_grid((y.float() / max(1, y.max())).unsqueeze(1), nrow=4),
    "batch_mask.png")
```

Then **overlay** (`vis[:, 0] = torch.where(y > 0, 1.0, vis[:, 0])`, save again). A side-by-side will
not reveal a 1-pixel misalignment or a transposed mask; an overlay will.

**2. Assert dtype and range after every stage.** Put these in the Dataset temporarily:

```python
assert img.dtype == np.float32, img.dtype
assert np.isfinite(img).all()
assert -6 < img.mean() < 6 and 0.2 < img.std() < 5, (img.mean(), img.std())
assert mask.dtype in (np.uint8, np.int64), mask.dtype
assert set(np.unique(mask)).issubset(ALLOWED_IDS), np.unique(mask)
```

The mask-class assertion catches: bilinear-interpolated masks, an unremapped label file where
classes are 0/38/75/113 instead of 0/1/2/3, and a padding fill of 0 colliding with a real class.

**3. Round-trip the tiler with an identity model.**

```python
img = np.random.rand(3, 1731, 2049).astype(np.float32)   # deliberately not divisible
rec = predict_scene(img, model_fn=lambda p: p, tile=512, stride=256, n_classes=3)
assert np.allclose(rec, img, atol=1e-5), np.abs(rec - img).max()
```

Use non-square, indivisible dimensions - a 2048x2048 test image passes with almost any broken
implementation.

**4. Verify inference normalization equals training normalization.** Store the stats *inside* the
checkpoint and have the inference script read them from there, so they cannot drift; if they live in
a sidecar file, assert equality of the loaded dicts at inference startup.

**5. Overfit 4 samples to ~0 loss.** With augmentation off. If the model cannot memorize 4 tiles,
the bug is in the data path (misaligned mask, wrong loss target dtype, ignore_index eating
everything), not in the architecture or LR.

**6. Count label pixels across the whole training set**, print the per-class fraction. A class at
0.0000 is absent - either genuinely, or because your remap dropped it.

**7. Flag constant tiles.** `img.std() < 1e-6` means an all-nodata tile reached training; those go
NaN under per-tile normalization and poison the whole batch.
