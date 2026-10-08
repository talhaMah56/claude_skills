---
name: medical-imaging-io
description: Load, convert, verify, and preprocess medical imaging data (DICOM, NIfTI, NRRD) for PyTorch/MONAI deep learning, and de-identify it safely. Use this when working with .dcm / DICOMDIR / .nii / .nii.gz / .mha / .nrrd files, pydicom, nibabel, SimpleITK, dcm2niix, or MONAI transforms; when converting a DICOM series to NIfTI; when a CT volume's intensities look wrong or Hounsfield-unit windowing / RescaleSlope / RescaleIntercept is involved; when volumes appear flipped or mirrored, or RAS vs LPS / affine / qform / sform orientation is in question; when resampling anisotropic voxel spacing to isotropic or writing Spacingd/Orientationd/ScaleIntensityRanged pipelines; when building a 3D segmentation or classification dataloader for CT/MRI/PET; and whenever PHI, de-identification, anonymization, defacing, burned-in annotations, IRB, or HIPAA come up in an imaging context.
---

# Medical Imaging I/O for Deep Learning

Scope: getting pixels off disk with the correct geometry, the correct intensity units, and no PHI. This is where most medical DL bugs are born, and almost all of them are *silent*.

## The five silent killers

1. Using stored pixel values as Hounsfield Units (no `RescaleSlope`/`Intercept`).
2. Sorting a DICOM series by `InstanceNumber` and getting a shuffled or reversed volume.
3. A left-right flip from LPS/RAS confusion, invisible on a brain scan, fatal for laterality labels.
4. Resampling a label map with linear interpolation, silently deleting thin structures.
5. Shipping PHI because you deleted `PatientName` and thought that was de-identification.

---

## DICOM with pydicom

```python
import pydicom
import numpy as np

ds = pydicom.dcmread("slice.dcm")                       # full read
hdr = pydicom.dcmread("slice.dcm", stop_before_pixels=True)  # headers only, ~100x faster

print(ds.Modality, ds.Rows, ds.Columns)
raw = ds.pixel_array          # ndarray, shape (Rows, Columns), STORED values
```

**Compressed transfer syntaxes.** `ds.pixel_array` raises `NotImplementedError` on JPEG baseline/lossless, JPEG-LS and JPEG2000 unless a decoder is installed. (RLE Lossless is the exception — pydicom decodes it natively, just slowly.) Install `pip install "pydicom[pixeldata]"` (pydicom ≥ 3) or explicitly `pylibjpeg pylibjpeg-libjpeg pylibjpeg-openjpeg`, or `python-gdcm`. Check `ds.file_meta.TransferSyntaxUID` before you blame your code.

**pydicom 2 vs 3 import path.** The rescale/VOI helpers moved:

```python
try:                                    # pydicom >= 3.0
    from pydicom.pixels import apply_modality_lut, apply_voi_lut
except ImportError:                     # pydicom 2.x
    from pydicom.pixel_data_handlers.util import apply_modality_lut, apply_voi_lut
```

### Tags that actually matter

| Tag | Meaning | Trap |
|---|---|---|
| `Modality` | CT / MR / PT / CR / DX / US | Gates whether HU semantics apply at all |
| `PixelSpacing` | `[row_mm, col_mm]` = **[y, x]** | Row spacing first. Reversing it silently transposes anisotropic in-plane geometry |
| `SliceThickness` | Reconstructed slab thickness | **Not** the slice-to-slice distance. Overlapping recons: thickness 5.0, spacing 3.0 |
| `SpacingBetweenSlices` | Center-to-center distance | Often absent, sometimes negative, sometimes wrong. Compute from `ImagePositionPatient` instead |
| `ImagePositionPatient` (IPP) | (x,y,z) mm of voxel (0,0) in patient LPS | The ground truth for slice location |
| `ImageOrientationPatient` (IOP) | 6 floats: row dir cosines + col dir cosines | Gives the slice normal via cross product |
| `RescaleSlope` / `RescaleIntercept` | Linear map to real units | See below |
| `PhotometricInterpretation` | `MONOCHROME1` / `MONOCHROME2` | MONOCHROME1 means **inverted** (high value = dark). Common in CR/DX/mammo |
| `PixelRepresentation` | 0 unsigned, 1 two's-complement signed | Wrong handling wraps −1000 HU to +64000 |
| `SeriesInstanceUID` | Series identity | The only reliable way to group files in a messy folder |

### The rescale trap

Stored pixel values are **not** Hounsfield Units. The DICOM modality LUT is:

```
real_value = stored_value * RescaleSlope + RescaleIntercept
```

For CT this is almost always `slope=1, intercept=-1024` (occasionally `-1000`), so a volume that looks like it ranges 0..4095 is really −1024..3071 HU. If you skip it, every HU threshold, every window preset, and every published intensity range you copy from a paper is wrong — and nothing errors. Air will sit at ~24 instead of −1000.

```python
hu = apply_modality_lut(ds.pixel_array, ds).astype(np.float32)
# equivalently, and safe when the tags are absent:
slope = float(getattr(ds, "RescaleSlope", 1.0))
inter = float(getattr(ds, "RescaleIntercept", 0.0))
hu = ds.pixel_array.astype(np.float32) * slope + inter
```

Prefer `apply_modality_lut` — it also handles the non-linear `ModalityLUTSequence` case that a manual multiply gets wrong.

Notes by modality: **CT** → HU, use it. **PET** → rescale gives Bq/mL, not SUV; SUV needs `PatientWeight`, injected dose, and decay correction from the radiopharmaceutical sequence — do not hand-roll it, use a validated converter. **MR** → slope/intercept usually 1/0 and the result is still an arbitrary scanner unit; see the MRI section. **CR/DX** → use `apply_voi_lut(arr, ds)` (window/VOI LUT), then invert if `PhotometricInterpretation == "MONOCHROME1"`.

### A DICOM series is many files, and sorting is not optional

`InstanceNumber` is an unreliable spatial ordering because:

- It is scanner-assigned metadata, not geometry. Some vendors restart it per acquisition, some number in reverse of anatomical direction, some omit it entirely.
- A single folder frequently contains **multiple series**: scouts/localizers, dose reports, multiple reconstruction kernels, multi-echo MR, pre/post contrast phases. Their `InstanceNumber` ranges overlap and interleave.
- Multi-echo / multi-phase acquisitions have several images at the *same* spatial position with different `InstanceNumber`.
- Even when it is right, it does not tell you the physical z-spacing or whether the stack runs head-to-foot or foot-to-head.

Sort by projecting `ImagePositionPatient` onto the slice normal derived from `ImageOrientationPatient`. That is a physical coordinate, monotonic by construction:

```python
from pathlib import Path
import numpy as np, pydicom

def load_ct_series(folder, series_uid=None):
    slices = []
    for p in Path(folder).rglob("*"):
        if not p.is_file():
            continue
        try:
            d = pydicom.dcmread(str(p), stop_before_pixels=True)
        except pydicom.errors.InvalidDicomError:
            continue
        if series_uid and d.SeriesInstanceUID != series_uid:
            continue
        if not hasattr(d, "ImagePositionPatient") or not hasattr(d, "ImageOrientationPatient"):
            continue                      # dose reports, SR, presentation states
        slices.append((p, d))

    if len(slices) < 2:
        raise ValueError(f"found {len(slices)} image slices in {folder} — wrong folder, "
                         "wrong series_uid, or an enhanced multi-frame file (see below)")

    iop = np.asarray(slices[0][1].ImageOrientationPatient, dtype=float)
    normal = np.cross(iop[0:3], iop[3:6])          # unit slice normal in LPS

    def loc(d):
        return float(np.dot(np.asarray(d.ImagePositionPatient, dtype=float), normal))

    slices.sort(key=lambda t: loc(t[1]))

    locs = np.array([loc(d) for _, d in slices])
    diffs = np.diff(locs)
    if np.any(np.isclose(diffs, 0)):
        raise ValueError("duplicate slice positions: multi-echo/phase mixed in, split further")
    if not np.allclose(diffs, diffs[0], atol=1e-2):
        raise ValueError(f"non-uniform z-spacing (missing slices?): {np.unique(np.round(diffs,3))}")
    z_spacing = float(abs(diffs[0]))               # TRUST THIS, not SliceThickness

    vol = np.stack([
        pydicom.dcmread(str(p)).pixel_array for p, _ in slices
    ]).astype(np.float32)                          # (Z, Y, X)

    d0 = slices[0][1]
    vol = vol * float(getattr(d0, "RescaleSlope", 1.0)) + float(getattr(d0, "RescaleIntercept", 0.0))
    py, px = (float(v) for v in d0.PixelSpacing)   # [row, col] = [y, x]
    return vol, (z_spacing, py, px)
```

The two `raise` statements are the point of the function. Non-uniform diffs mean missing slices (a corrupt download, or a gap in the acquisition) — stacking anyway produces a geometrically wrong volume with no error. Duplicate positions mean you have more than one image per location and must split by `EchoTime`, `AcquisitionNumber`, `ImageType`, or `ContrastBolusAgent`.

Also check `d0.ImageOrientationPatient` is identical for all slices (gantry tilt or a mid-series reorientation invalidates the whole stack) and reject the series if any slice differs beyond ~1e-4.

The function above applies the *first* slice's `RescaleSlope`/`RescaleIntercept` to the whole stack. That is safe for CT, where they are constant across a series, but **not** for PET and for some Philips MR, where each slice carries its own scaling. Assert the pair is constant across slices, or rescale per slice before stacking.

**Enhanced/multi-frame DICOM** (a single file holding the whole volume, common on Philips and modern Siemens) does not have per-file `PixelSpacing`. It lives in `SharedFunctionalGroupsSequence` / `PerFrameFunctionalGroupsSequence`. Do not write a parser for this by hand — use SimpleITK or dcm2niix.

### When to just use SimpleITK

```python
import SimpleITK as sitk
reader = sitk.ImageSeriesReader()
ids = reader.GetGDCMSeriesIDs(folder)                       # one entry per series — use it
files = reader.GetGDCMSeriesFileNames(folder, ids[0])
reader.SetFileNames(files)
img = reader.Execute()          # geometry, sorting, rescale all handled
arr = sitk.GetArrayFromImage(img)   # (Z, Y, X) — REVERSED vs img.GetSize() (X, Y, Z)
spacing = img.GetSpacing()          # (X, Y, Z) — also reversed vs arr axes
```

The axis-order reversal between `GetSize()`/`GetSpacing()` and `GetArrayFromImage()` is the single most common SimpleITK bug. SimpleITK works in **LPS**.

---

## CT windowing

A CT volume spans roughly −1024 to +3071 HU. Min-max normalizing that whole range is wrong for two reasons:

1. **Dynamic range collapse.** Everything clinically interesting in an abdominal scan lives between −150 and +250 HU — about 10% of the range. Min-max squeezes all soft-tissue contrast into a tenth of your network's input scale.
2. **Non-determinism.** A single metal clip, dental filling, or table artifact at 3071 HU changes the max, which changes the scaling of *every other voxel* in that volume. Two scans of the same patient normalize differently. Your network sees a per-volume random contrast jitter it cannot correct for.

Windowing is a **fixed, dataset-independent, physically meaningful** affine map. The same window applied to any scanner from any vendor produces comparable numbers, so it transfers.

```python
def window(hu, level, width):
    lo, hi = level - width / 2.0, level + width / 2.0
    return (np.clip(hu, lo, hi) - lo) / (hi - lo)   # float32 in [0, 1]
```

| Preset | WL (center) | WW (width) | HU range |
|---|---|---|---|
| Brain parenchyma | 40 | 80 | 0 → 80 |
| Subdural / blood | 75 | 215 | −32 → 182 |
| Acute stroke (narrow) | 35 | 30 | 20 → 50 |
| Temporal bone | 600 | 2800 | −800 → 2000 |
| Soft tissue / mediastinum | 50 | 400 | −150 → 250 |
| Liver (narrow) | 60 | 160 | −20 → 140 |
| Lung | −600 | 1500 | −1350 → 150 |
| Bone | 400 | 1800 | −500 → 1300 |
| CTA / angiography | 300 | 600 | 0 → 600 |

These are approximate and vary by institution; treat them as starting points, not constants. The abdominal-organ segmentation default used by the MONAI/BTCV tutorials is `a_min=-175, a_max=250` (≈ WL 38 / WW 425), which is a reasonable generic soft-tissue window.

**Multi-window input.** A strong, cheap trick: stack 3 windows (e.g. brain / subdural / bone for head CT, or soft-tissue / lung / bone for chest) as 3 channels. You get the benefit of a pretrained 3-channel RGB backbone and you stop having to pick one window.

HU sanity anchors for debugging: air −1000, lung −700, fat −90, water 0, CSF 15, white matter 25, gray matter 40, muscle 45, liver 55, clotted blood 70, cancellous bone 350, cortical bone 1000+. If air in your volume is not near −1000, your rescale is wrong.

---

## MRI is not CT

MR intensities have **no physical unit and no cross-scanner meaning**. The same tissue in the same patient scanned on a 1.5T GE and a 3T Siemens produces completely different numbers; even the same scanner varies run to run with receiver gain and coil loading. There is no MR equivalent of a window preset. Consequences:

- **Never** apply a fixed intensity range to MR. Per-volume normalization is mandatory.
- Standard practice (BraTS and successors): z-score over **non-zero / in-brain voxels only**. Including the background zeros pulls the mean toward zero and makes the statistics depend on how much air is in the field of view — i.e. on the crop, not the anatomy.

```python
def znorm_brain(vol, mask=None):
    m = mask if mask is not None else (vol > 0)
    v = vol[m]
    return np.where(m, (vol - v.mean()) / (v.std() + 1e-8), 0.0).astype(np.float32)
```

MONAI equivalent: `NormalizeIntensityd(keys="image", nonzero=True, channel_wise=True)` — `channel_wise=True` matters when the 4 BraTS sequences are stacked as channels, since T1/T1ce/T2/FLAIR have unrelated scales.

- Alternatives: percentile clipping (clip to the 0.5th/99.5th percentile of in-mask voxels, then scale to [0,1]) is more robust to a single bright vessel or fat artifact than min-max. Nyúl/Udupa histogram matching standardizes across a dataset, but it must be fit on the training split only.
- **Bias field.** Low-frequency multiplicative intensity drift across the field of view. Correct with N4: `SimpleITK.N4BiasFieldCorrectionImageFilter` (run on a downsampled image and apply the log-bias field at full resolution — full-res N4 is slow), or ANTs `N4BiasFieldCorrection`. Do N4 **before** normalization: correcting the field after you have z-scored is fitting to numbers the bias already corrupted.
- Multi-sequence data must be co-registered before stacking as channels. Different sequences in the same session are usually close but not identical in geometry — check the affines, do not assume.

---

## NIfTI with nibabel

```python
import nibabel as nib, numpy as np

img = nib.load("scan.nii.gz")          # lazy: header only, no pixels read
print(img.shape, img.header.get_zooms())   # zooms = voxel size in mm (+ TR for 4D)
aff = img.affine                        # 4x4 voxel-index -> world (mm) in RAS

data = img.get_fdata(dtype=np.float32)  # loads EVERYTHING
```

- `get_fdata()` defaults to **float64**. A 512×512×400 int16 volume is 200 MB on disk and 840 MB as float64. Pass `dtype=np.float32` always.
- For one slice or a 4D timepoint, slice the proxy instead: `sl = np.asarray(img.dataobj[..., 40])`. `dataobj` reads only the requested block and still applies `scl_slope`/`scl_inter`. Raw unscaled values: `img.dataobj.get_unscaled()`.
- To crop while keeping geometry correct, use `img.slicer[20:200, :, :]` — it updates the affine. Slicing the ndarray does not, and your labels will no longer align.
- `nib.Nifti1Image(arr, affine)` then `nib.save(img, path)`. If you changed voxel size, also `img.header.set_zooms(...)` — or just build the affine correctly and let nibabel derive zooms.
- **qform vs sform.** NIfTI stores two affines. `img.affine` returns sform if `sform_code > 0`, else qform. If they disagree (common after ad-hoc header edits), different toolkits will place your volume differently. Check `img.header.get_qform(coded=True)` vs `get_sform(coded=True)` and make them consistent.
- `.nii.gz` decompression is a real dataloader bottleneck: gunzip is single-threaded per file and a 300 MB volume costs ~1–3 s of pure CPU. With 8 workers you will saturate CPU before the GPU is busy. Convert once to uncompressed `.nii`, `.npy` memmaps, or use MONAI `PersistentDataset` / `CacheDataset`.

---

## Orientation: RAS vs LPS

Both are world coordinate conventions naming the **positive** direction of each axis:

- **LPS** = +x → patient Left, +y → Posterior, +z → Superior. Used by **DICOM**, ITK, SimpleITK, NRRD (usually).
- **RAS** = +x → Right, +y → Anterior, +z → Superior. Used by **NIfTI**, nibabel, FreeSurfer, MONAI's canonical target.

Converting LPS ↔ RAS negates the first two axes. Get this wrong in one direction only and you have a **left-right mirror**.

State it plainly: **a left-right flip on a brain MRI is essentially invisible.** The brain is grossly symmetric; a flipped volume looks like a perfectly normal brain. But if your task involves laterality — left vs right hippocampal atrophy, side of stroke, left vs right kidney, "lesion in the right upper lobe" — a flip silently converts a fraction of your training labels into the opposite class. Metrics degrade a few points and you will blame the model. Same for any asymmetric anatomy: liver/spleen, cardiac chambers, situs.

```python
# Reorient to closest canonical RAS.
img_ras = nib.as_closest_canonical(img)
print(nib.aff2axcodes(img.affine), "->", nib.aff2axcodes(img_ras.affine))  # e.g. ('L','A','S') -> ('R','A','S')
```

`as_closest_canonical` only transposes and flips axes — it never interpolates, so it is lossless and safe to run on label maps. It maps to the nearest axis-aligned RAS; for an obliquely acquired volume the result is still oblique, just closest-to-RAS.

For an arbitrary target orientation:

```python
from nibabel.orientations import (io_orientation, axcodes2ornt, ornt_transform,
                                  apply_orientation, inv_ornt_aff)
src = io_orientation(img.affine)
dst = axcodes2ornt(("R", "A", "S"))
xf  = ornt_transform(src, dst)
arr = apply_orientation(img.get_fdata(dtype=np.float32), xf)
# apply_orientation ONLY permutes/flips the array. The affine must be updated
# too, or the new array is silently paired with the old geometry:
new_affine = img.affine @ inv_ornt_aff(xf, img.shape)   # shape = the ORIGINAL shape
img_ras = nib.Nifti1Image(arr, new_affine)
```

**Verification rule:** apply the identical reorientation to image and label with the same code path, then assert the label's nonzero centroid still sits inside the image's foreground. And once, by hand, for one subject: verify laterality against a known asymmetric ground truth (the liver is on the patient's right, the heart apex points left). Do not verify a flip by looking at a brain.

Anatomical fiducials beat eyeballing. If your dataset has any laterality metadata (a report saying "right-sided lesion"), use it as an automated check across the whole dataset.

---

## Voxel spacing and resampling

Real clinical volumes are often strongly **anisotropic**: 0.7 × 0.7 × 5.0 mm is a routine abdominal CT. A 3×3×3 convolution treats all three axes as equivalent, so its receptive field is 2.1 × 2.1 × 15 mm — it "sees" 7× further in z than in-plane, and a pooling stack compounds this. Learned filters end up encoding the acquisition protocol rather than anatomy, so a model trained on 5 mm slices generalizes poorly to 1 mm ones.

Two valid responses:

| Approach | When it wins |
|---|---|
| Resample to isotropic (e.g. 1×1×1 mm) | Spacing is mildly anisotropic (ratio < 3), you need geometric consistency across a mixed-protocol dataset, and you have the memory |
| Keep anisotropy, use anisotropic architecture | Ratio ≥ 3 (e.g. 5 mm slices). Interpolating 0.7 → 5 mm z invents 7× data that is not there; better to use anisotropic patch sizes and skip z-pooling in early stages, as nnU-Net does |

nnU-Net's concrete heuristic, worth copying: target spacing = **median** spacing per axis across the training set; if the axis anisotropy ratio exceeds **3**, use the **10th percentile** spacing for the low-resolution axis instead of the median, and resample that axis with nearest-neighbour (order 0) while using third-order spline in-plane. This avoids fabricating smooth detail along a direction where none was measured.

```python
from nibabel.processing import resample_to_output
img_iso  = resample_to_output(img, voxel_sizes=(1.0, 1.0, 1.0), order=3)  # image: cubic spline
lbl_iso  = resample_to_output(lbl, voxel_sizes=(1.0, 1.0, 1.0), order=0)  # label: NEAREST
```

**Interpolation order rules — the non-negotiable part:**

- **Labels: `order=0` (nearest), always.** Linear interpolation of an integer label map produces values like 1.4 between class 1 and 2; casting back to int yields class 1, class 2, or 0 depending on rounding, inventing boundary classes that never existed. Worse, downsampling a 2-voxel-thick structure with linear interpolation followed by rounding can drop it entirely — your ground truth loses vessels, small lesions, and thin walls with no error and no warning. Always assert `set(np.unique(lbl_out)) <= set(np.unique(lbl_in))` after resampling.
- If you need sub-voxel-accurate label boundaries, one-hot encode, resample each channel linearly, then argmax. Costs C× memory; worth it for small-structure segmentation.
- **Images: `order=1` (linear) or `order=3` (spline).** Spline is sharper but **overshoots** at high-contrast edges (air/bone in CT), producing HU values outside the physical range — e.g. −1300 next to bone. Clip after: `np.clip(out, hu_min, hu_max)`. Especially clip before applying a window, or the overshoot leaks into your normalized values.
- Resample **before** intensity normalization when normalization statistics are computed per-volume, so the statistics are computed on the data the network actually sees.

**Preserving the affine.** Resampling changes voxel size, so the affine's rotation/scale block and origin both change. `nibabel.processing.resample_to_output` and MONAI's `Spacingd` handle this. If you resample with `scipy.ndimage.zoom` you must recompute the affine yourself — and this is exactly where image and label drift apart by half a voxel. Prefer library resamplers. To force a label onto an image's exact grid: `nibabel.processing.resample_from_to(lbl, img, order=0)`.

Ordering constraint: **reorient → resample → crop → normalize**. Reorienting after resampling is fine but you pay for interpolation on an axis you then transpose; cropping before resampling changes what "foreground" means at the new spacing; normalizing before cropping computes statistics over background air you are about to discard.

---

## De-identification and PHI — mandatory

Treat this as a hard gate before data leaves a secure environment, appears in a notebook, gets uploaded to a cloud GPU, or goes into a figure. **Ad-hoc tag deletion is not de-identification.** A compliant process follows a named profile.

**The standard.** DICOM PS3.15 Annex E, the *Basic Application Level Confidentiality Profile*, plus option sets (Retain Longitudinal Temporal Dates, Retain Device Identity, Clean Pixel Data, Clean Descriptors, Retain Safe Private, …). Table E.1-1 enumerates every tag and its required action (X = remove, Z = zero, D = replace with dummy, U = replace UID consistently). Use an implementation of it, not your own list.

**Tags carrying obvious PHI:** `PatientName`, `PatientID`, `OtherPatientIDs`, `PatientBirthDate`, `PatientAddress`, `PatientTelephoneNumbers`, `AccessionNumber`, `StudyID`, `StudyDate`/`StudyTime`, `SeriesDate`, `AcquisitionDate`, `InstitutionName`, `InstitutionAddress`, `ReferringPhysicianName`, `PerformingPhysicianName`, `OperatorsName`, `StationName`, `DeviceSerialNumber`.

**The ones people miss:**

- **Free-text descriptors.** `StudyDescription`, `SeriesDescription`, `ImageComments`, `ProtocolName`, `RequestedProcedureDescription`, and structured-report content routinely contain names, MRNs, and "MR BRAIN — SMITH, JOHN".
- **Private tags.** Vendor blocks (Siemens CSA headers, GE private groups) can carry patient identifiers, exam notes, and full protocol dumps. `ds.remove_private_tags()` removes odd-group elements — do this, but understand it also destroys diffusion b-vectors and other tags you may need, so extract what you need first.
- **UIDs.** `StudyInstanceUID`, `SeriesInstanceUID`, `SOPInstanceUID`, `FrameOfReferenceUID` are not PHI themselves but are linkable back to the PACS. Replace them with a **consistent** mapping (`pydicom.uid.generate_uid()`), not by deletion — deleting `FrameOfReferenceUID` breaks the link between an image and its RTSTRUCT/SEG.
- **Dates.** Deleting all dates destroys longitudinal intervals. The right move is a **consistent per-patient offset** (shift every date for one patient by the same random number of days), which preserves follow-up intervals while removing the real calendar date. Ages over 89 are separately identifying under HIPAA Safe Harbor.
- **Burned-in pixel PHI.** Text rendered *into the image data*. Endemic in **ultrasound**, **secondary capture**, PACS screenshots, scanned documents, dose-report images, and older fluoroscopy/mammography. The `BurnedInAnnotation` (0028,0301) tag is **not trustworthy** — it is frequently absent, or present and set to `NO` when there is clearly text in the corner. You must handle this with pixel-level tooling (OCR-based detection plus masking) or by excluding those SOP classes entirely. No header scrub touches it.

```python
import pydicom
from pydicom.uid import generate_uid

ds = pydicom.dcmread(p)
ds.remove_private_tags()                       # necessary, NOT sufficient
for tag in ("PatientName", "PatientID", "PatientBirthDate", "PatientAddress",
            "AccessionNumber", "InstitutionName", "ReferringPhysicianName",
            "StudyDescription", "SeriesDescription", "ImageComments", "StationName"):
    if tag in ds:
        delattr(ds, tag)
ds.PatientIdentityRemoved = "YES"
ds.DeidentificationMethod = "PS3.15 Annex E Basic Profile (tool: <name/version>)"
# ...plus consistent UID remapping, date shifting, and pixel inspection.
```

Treat that snippet as a *starting point that would fail an audit on its own*. Use real tooling:

| Tool | Use |
|---|---|
| **RSNA CTP** (Clinical Trials Processor) | The reference implementation of PS3.15 profiles; scriptable anonymizer, used for real multi-site studies |
| **`dicognito`** (pip) | Pragmatic Python DICOM anonymizer, consistent pseudonyms across a study |
| **`deid`** (Stanford / `pydicom/deid`, `pip install deid`) | Header recipes **plus** burned-in pixel PHI detection/scrubbing |
| **`dicom-anonymizer`** (pip; imports as `dicomanonymizer`) | Profile-driven PS3.15 tag actions, usable as a library or CLI |
| **`dcm2niix`** | Drops most PHI tags during conversion — a side effect, not a compliance guarantee |

**Defacing.** A head MRI or CT is a full 3D scan of the face. Volume-render it and you get a recognizable photograph; face-recognition matching against public photos has been demonstrated. If you release head imaging, deface it: **`pydeface`** (FSL-based), FreeSurfer **`mideface`** (or the older `mri_deface`), AFNI **`afni_refacer_run`** (refacing, replaces rather than deletes the face), **`quickshear`**. Run defacing *after* any registration or brain extraction that needs the face, and check afterwards that the mask did not clip cerebellum or temporal lobes — defacing tools do sometimes eat brain tissue, which corrupts your task silently.

**Governance overrides all of the above.** Your IRB protocol, the data use agreement, and (in the US) HIPAA Safe Harbor / Expert Determination, or GDPR in the EU, define what is permitted for *your* dataset. A technically thorough scrub does not authorize redistribution. If a DUA says "may not be shared outside the approved institution", no amount of de-identification changes that. When in doubt, ask the data steward, not the code.

---

## dcm2niix: use it

`dcm2niix` is the de facto DICOM→NIfTI converter (also embedded in `dcm2bids`, HeuDiConv, MRIcroGL). Use it rather than hand-rolling because it already handles: series splitting, correct slice ordering including reversed/interleaved acquisitions, gantry tilt correction, vendor-specific quirks (Siemens mosaic EPI, Philips scaling, GE private headers), enhanced multi-frame DICOM, diffusion bvec/bval extraction **in the correct reoriented frame**, and complex/phase/magnitude separation. Each of those is a multi-day bug if you write it yourself; the Siemens mosaic and bvec-reorientation cases in particular are known to be gotten wrong by hand-rolled code.

```bash
dcm2niix -z y -f "%p_%s" -o /out /in/dicom_dir
#  -z y     gzip output (.nii.gz)
#  -f       filename template: %p protocol, %s series number, %i patient ID (avoid %i for shared data)
#  -b y     write BIDS JSON sidecar (default y) — keeps EchoTime, RepetitionTime, slice timing
#  -i y     ignore derived/localizer/2D images
#  -m y     merge series that differ only in trivial ways (use with care)
#  -w 1     overwrite name conflicts instead of adding suffixes
```

Read the emitted `.json` sidecar — it is the only place a lot of acquisition metadata survives. dcm2niix outputs NIfTI in **RAS**, converting from DICOM LPS for you.

Labels that come as **RTSTRUCT** (contour polygons) or **DICOM SEG** need separate handling: `dcmrtstruct2nii`, `rt-utils`, or `pydicom-seg`. RTSTRUCT contours are stored in patient mm coordinates and must be rasterized onto the image grid — the resulting mask's alignment depends on the *image* series you rasterize against, so convert the image first and rasterize onto it.

---

## MONAI dictionary pipeline

MONAI is the domain-standard transform library. Dictionary transforms (`*d` suffix) apply the same spatial operation to image and label with a single random state, which is why they exist — the alternative is silently desynchronized augmentation.

```python
from monai.transforms import (
    Compose, LoadImaged, EnsureChannelFirstd, Orientationd, Spacingd,
    ScaleIntensityRanged, CropForegroundd, RandCropByPosNegLabeld,
    RandFlipd, RandShiftIntensityd, EnsureTyped,
)

K = ["image", "label"]

train_tf = Compose([
    LoadImaged(keys=K, image_only=False),                    # reads NIfTI/DICOM/NRRD, keeps meta
    EnsureChannelFirstd(keys=K),                             # (H,W,D) -> (1,H,W,D)
    Orientationd(keys=K, axcodes="RAS"),                     # BEFORE Spacingd
    Spacingd(keys=K, pixdim=(1.5, 1.5, 2.0),
             mode=("bilinear", "nearest")),                  # image bilinear, LABEL NEAREST
    ScaleIntensityRanged(keys="image", a_min=-175, a_max=250,
                         b_min=0.0, b_max=1.0, clip=True),   # HU window, image only
    CropForegroundd(keys=K, source_key="image", allow_smaller=True),
    RandCropByPosNegLabeld(
        keys=K, label_key="label", spatial_size=(96, 96, 96),
        pos=1, neg=1, num_samples=4, image_key="image", image_threshold=0,
    ),
    RandFlipd(keys=K, spatial_axis=[0], prob=0.5),           # see laterality warning below
    RandShiftIntensityd(keys="image", offsets=0.10, prob=0.5),
    EnsureTyped(keys=K),
])
```

**Why this order, and what breaks if you reorder:**

1. `LoadImaged` first — everything downstream needs the meta dict (affine, spacing) it attaches. In MONAI ≥ 1.3 `image_only` defaults to `True`, returning a `MetaTensor` that still carries the affine; pass `image_only=False` only if you want the legacy separate `_meta_dict` entries.
2. `EnsureChannelFirstd` before any spatial transform. `Spacingd` and the crops assume channel-first `(C, H, W, D)`. On a bare `(H, W, D)` volume they will treat H as the channel axis and mangle geometry without erroring.
3. `Orientationd` **before** `Spacingd`. `pixdim=(1.5, 1.5, 2.0)` names *axes*, and which anatomical axis is index 2 depends on orientation. Reorder these and a coronally-stored volume gets 2.0 mm applied to its anterior-posterior axis instead of superior-inferior — a subtle, dataset-dependent corruption.
4. `Spacingd` **before** `CropForegroundd` and before the random crop, because `spatial_size=(96,96,96)` is in voxels. At 0.7 mm that patch covers 67 mm; at 1.5 mm it covers 144 mm. Cropping first makes your patch's physical field of view vary per subject.
5. `ScaleIntensityRanged` on `"image"` only. Applying an intensity transform to `"label"` destroys the class indices. This is the most common copy-paste bug in MONAI pipelines — check the `keys` of every intensity transform.
6. `CropForegroundd` after windowing: it thresholds on intensity, and the threshold means something different in HU than in [0,1]. `source_key="image"` makes the crop box come from the image and be applied to both. Set `allow_smaller` explicitly — the default changed across MONAI versions, and the wrong value either errors on small volumes or silently returns patches below `spatial_size`.
7. Random crops after all deterministic spatial work, so caching (`CacheDataset`) can memoize everything up to the first random transform. Put the deterministic transforms first or caching buys you nothing.
8. `RandFlipd` on the left-right axis is standard for symmetric-task brain segmentation but is **wrong for any laterality-dependent label** and for abdominal organs (it teaches the network that livers appear on both sides). Decide deliberately.

For MRI, replace `ScaleIntensityRanged` with `NormalizeIntensityd(keys="image", nonzero=True, channel_wise=True)`.

Validation pipeline: identical, minus the random transforms and usually minus the random crop (use `SlidingWindowInferer` at full resolution instead). If validation preprocessing differs from training preprocessing in anything except randomness, your validation number is measuring the wrong thing.

Dataset choice: `CacheDataset` (RAM, fastest, needs the deterministic prefix to fit in memory), `PersistentDataset` (caches to disk — right choice for volumes too big for RAM), plain `Dataset` (no caching). `SmartCacheDataset` for datasets larger than RAM with epoch-wise replacement.

---

## Windows specifics

- **DataLoader workers use `spawn`, not `fork`.** Every worker re-imports the training module and unpickles the transform chain, so the entry point must sit behind `if __name__ == "__main__":` or workers respawn recursively. Consequence for MONAI: a `CacheDataset`'s cache is built in the parent and then **copied into every worker** — peak RAM is roughly `cache_size × (num_workers + 1)`. On 3D volumes that OOMs fast; use `PersistentDataset` (disk cache, each worker reads its own items) instead of raising `cache_rate`.
- Worker startup costs seconds, not milliseconds. Set `persistent_workers=True` and keep `num_workers` at or below the physical core count; more workers is often slower here than on Linux.
- **MAX_PATH (260 chars).** DICOM trees keyed by UID (`.../1.2.840.113619.../1.2.840.113619....dcm`) overflow it routinely, and the failure surfaces as `FileNotFoundError` on a file that visibly exists. Enable `LongPathsEnabled` in the registry, or prefix absolute paths with `\\?\`. Flatten/rename on ingest where you can.
- Dataset prep scripts written for Linux often use symlinks; creating one on Windows needs Developer Mode or admin. Copy or use a manifest of paths instead.
- `dcm2niix` has no pip wheel that ships the binary — get it from conda-forge (`conda install -c conda-forge dcm2niix`) or the MRIcroGL bundle, and quote paths containing spaces.

---

## Pre-training checklist

Geometry
- [ ] Every volume and its label have matching shape **and** matching affine (compare with `np.allclose`, tolerance ~1e-3, not `==`).
- [ ] Reorientation applied identically to image and label; laterality verified once by hand against known asymmetric anatomy — not by looking at a brain.
- [ ] z-spacing derived from `ImagePositionPatient` differences, not `SliceThickness`.
- [ ] Spacing distribution across the dataset printed and inspected; target spacing chosen from it, not guessed.
- [ ] After resampling: `np.unique(label)` unchanged; per-class voxel counts logged before and after (a class that lost >20% of its voxels was destroyed by interpolation).

Intensity
- [ ] Rescale slope/intercept applied. Air sits near −1000 HU on CT.
- [ ] Window chosen for the task and applied identically at train and inference time.
- [ ] MRI normalized per-volume over a brain/nonzero mask; no fixed intensity range anywhere.
- [ ] `nan`/`inf` check on every volume (`np.isfinite(vol).all()`) — a single NaN destroys a training run via the loss.
- [ ] Label values are contiguous integers starting at 0, and match what the loss expects.

Data hygiene
- [ ] Splits are by **patient**, never by slice or by volume. The same patient in train and val leaks and inflates Dice by several points.
- [ ] Duplicate patients across sources checked (public datasets overlap: e.g. TCIA collections re-appear inside aggregated benchmarks).
- [ ] Series filtered to the intended acquisition: no localizers, dose reports, or wrong contrast phase.

Safety
- [ ] De-identification run with a named profile/tool, recorded in `DeidentificationMethod`.
- [ ] Private tags removed; UIDs consistently remapped; dates shifted, not blindly deleted.
- [ ] Burned-in pixel PHI checked for ultrasound / secondary capture / screenshots — the `BurnedInAnnotation` tag is not evidence.
- [ ] Head imaging defaced, and the defacing verified not to have removed brain.
- [ ] IRB approval and DUA terms confirmed to cover the intended use, storage location, and any cloud compute.
