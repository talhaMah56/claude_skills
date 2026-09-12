---
name: geospatial-raster-io
description: Read, write, align, and normalize geospatial raster and vector data for ML training. Use when working with GeoTIFF/.tif/.tiff, COG, NetCDF/.nc, HDF5/.h5/.hdf, Zarr, shapefiles/.shp/GeoJSON, or the libraries rasterio, xarray, rioxarray, geopandas, fiona, pyproj, GDAL, gdalwarp. Triggers on tasks like "load this GeoTIFF", "tile a raster into patches", "reproject to EPSG:32633", "why is my normalization broken", "nodata / -9999 / NaN in my imagery", "align my label mask to my image", "rasterize these polygons", "burn shapefile into a label raster", "Sentinel-2 bands / SCL cloud mask / BOA offset", "Landsat Collection 2 scale factor", "QA_PIXEL bitmask", "open_mfdataset time stack", "chunk a NetCDF with dask", "compute per-band mean/std for a satellite dataset", or "install GDAL/rasterio on Windows". Also use when a model trains but predictions have black borders, when class counts don't match the legend, or when two rasters "look aligned" but produce garbage.
---

# Geospatial Raster I/O for ML

## The four failure modes that cost weeks

1. **Nodata sentinels enter your statistics.** `-9999` in a mean makes it negative; `0` biases reflectance low; NaN propagates (this is the *lucky* case — it's loud).
2. **Transform is dropped during cropping.** Your tiles are numpy arrays with no georeference. Predictions can never be written back to a map.
3. **Bilinear resampling on a label raster.** Class 3 next to class 7 becomes class 5. No error. Your confusion matrix has classes that don't exist in the legend.
4. **Two rasters that "look aligned."** Same shape, same plot extent, different CRS or a half-pixel transform offset. Labels are shifted by 1–2 pixels everywhere; mIoU caps at ~0.6 and you blame the model.

---

## rasterio: GeoTIFF I/O

```python
import rasterio
from rasterio.windows import Window

with rasterio.open("scene.tif") as src:
    print(src.crs)         # CRS.from_epsg(32633)  -- may be None!
    print(src.transform)   # Affine(10.0, 0.0, 399960.0, 0.0, -10.0, 4600020.0)
    print(src.count, src.width, src.height, src.dtypes, src.nodata)
    print(src.bounds, src.res, src.block_shapes)

    all_bands = src.read()            # (count, H, W)  -- band-first, NOT HWC
    red       = src.read(4)           # (H, W), 1-INDEXED bands
    rgb       = src.read([4, 3, 2])   # (3, H, W)
```

**Band indices are 1-based.** `src.read(0)` is an error, not band 0.

### Windowed reads

`Window(col_off, row_off, width, height)` — **column first, then row**, the opposite of numpy `[row, col]`. This transposition is the single most common tiling bug.

```python
win = Window(col_off=1024, row_off=512, width=256, height=256)
patch = src.read(window=win)                  # (count, 256, 256)
win_transform = src.window_transform(win)     # CARRY THIS
# equivalently: rasterio.windows.transform(win, src.transform)
```

- A window extending past the raster edge is **clipped silently** — you get a smaller array, and `np.stack` on your batch then fails far away from the cause. Either check `win.width`/`patch.shape` or use `src.read(window=win, boundless=True, fill_value=0)` and mask the fill.
- Windowed reads are only fast on **tiled** TIFFs. On a striped TIFF, a 256×256 window decodes full-width strips. Check `src.block_shapes`: `[(1, 10980)]` means striped (slow), `[(512, 512)]` means tiled. Iterate `src.block_windows(1)` to read block-aligned.
- Opening the file once per patch in a `Dataset.__getitem__` is fine, but a `DatasetReader` handle **cannot be shared across DataLoader workers** (GDAL handles are not fork/spawn safe). Open lazily inside `__getitem__` or in `worker_init_fn`.

### Writing: copy the profile, don't build one

```python
import numpy as np

with rasterio.open("scene.tif") as src:
    profile = src.profile.copy()      # driver, dtype, nodata, w, h, count, crs, transform, + creation opts
    data = src.read(masked=True)

out = compute(data).astype("float32")
profile.update(
    dtype="float32", count=out.shape[0], nodata=np.nan,
    compress="deflate", predictor=3,          # predictor=2 for ints, 3 for floats, omit for uint8
    tiled=True, blockxsize=512, blockysize=512,  # must be multiples of 16
    BIGTIFF="IF_SAFER",
)
with rasterio.open("out.tif", "w", **profile) as dst:
    dst.write(out)                             # 3D (count,H,W), or dst.write(arr2d, 1)
```

Copy-and-mutate because the profile carries `crs`, `transform`, `nodata`, and driver creation options you would otherwise have to reconstruct exactly. Hand-building a profile is how a raster ends up with `crs=None` and a default identity transform.

Three hard constraints:
- **`nodata` must be representable in `dtype`.** `nodata=-9999` with `dtype="uint16"` raises (or, in older stacks, wraps to 55537 — a valid-looking value).
- **Cast the array yourself.** Do not rely on rasterio to convert float32 → uint16 for you; be explicit, and clip before casting or you get wraparound, not saturation.
- **The profile also carries the source `driver`.** Sentinel-2 SAFE bands are JPEG2000 (`driver="JP2OpenJPEG"`); copy that profile, name the output `out.tif`, and you have written a JP2 with a `.tif` extension — or failed outright on `tiled`/`predictor`/`BIGTIFF`, which are GTiff-only creation options. Set `driver="GTiff"` explicitly whenever the source is not already a GeoTIFF.

To emit a COG directly, use `driver="COG"` (rasterio ≥1.3, GDAL ≥3.1): it builds the overviews for you and takes `blocksize=512`, `overview_resampling="average"` — `blocksize` singular, not the GTiff `tiled`/`blockxsize`/`blockysize` trio.

---

## Nodata: the silent corruption source

```python
arr = src.read(1)                 # nodata pixels are just numbers: -9999, 0, 65535
arr.mean()                        # meaningless
```

### Get a real mask

```python
data = src.read(masked=True)      # np.ma.MaskedArray; data.mask is True where INVALID
valid = ~data.mask                # numpy convention is inverted from "valid mask"
```

**`masked=True` only produces a mask if the file declares nodata** (`src.nodata is not None`) or has an internal mask / alpha band. If `src.nodata is None`, `masked=True` returns `mask=False` everywhere and you have silently done nothing. Always check:

```python
if src.nodata is None:
    # you must supply the sentinel from the product spec / metadata
    mask = (arr == KNOWN_SENTINEL)
```

Other traps:
- **NaN never compares equal.** `arr == np.nan` is all-False. Use `np.isnan(arr)`. And `src.nodata` for a NaN-nodata file reads back as `nan`, so `arr == src.nodata` is also all-False.
- `src.dataset_mask()` returns uint8 0/255 (255 = valid) combining nodata + alpha + internal masks. `src.read_masks(1)` is the per-band version.
- Nodata does not survive arithmetic. `a / b` on masked arrays keeps the mask; on plain arrays it does not.
- In the loss: index with the valid mask, never `nan_to_num`. Replacing nodata with 0 teaches the model that 0 is a real observation.

### Dataset statistics over valid pixels only

Two-pass or streaming, always in **float64** accumulators. `sqrt(E[x²] − E[x]²)` in float32 on Sentinel-2 DNs (mean ≈ 2000, std ≈ 500) loses most of the significant digits to cancellation.

```python
import numpy as np, rasterio

def band_stats(paths, nbands):
    n   = np.zeros(nbands, dtype=np.float64)
    s   = np.zeros(nbands, dtype=np.float64)
    ss  = np.zeros(nbands, dtype=np.float64)
    for p in paths:
        with rasterio.open(p) as src:
            a = src.read(masked=True).astype(np.float64)   # (B,H,W)
            for b in range(nbands):
                v = a[b].compressed()                       # 1D, valid pixels only
                n[b]  += v.size
                s[b]  += v.sum()
                ss[b] += (v ** 2).sum()
    mean = s / n
    var  = ss / n - mean ** 2
    return mean, np.sqrt(np.maximum(var, 0.0))              # clamp: cancellation can go negative
```

Sanity gates before you trust these: `n` must be well below `len(paths) * H * W` if there is any nodata; `mean` must lie inside the physical range of the product (e.g. 0–1 for reflectance, 0–10000 for raw S2 DN); no NaN or inf.

For heavy-tailed sensors (SAR, radiance), z-scoring on mean/std is often worse than **per-band 2nd/98th percentile clipping** computed on a random sample of valid pixels. Reservoir-sample ~10⁷ valid pixels and take `np.percentile`; a full-dataset exact percentile is rarely worth the pass.

---

## CRS and reprojection

A CRS says what the numbers in `transform` mean. `EPSG:4326` = lat/lon degrees (WGS84). `EPSG:326xx`/`327xx` = UTM zone xx north/south, metres — this is what Sentinel-2 tiles ship in. `EPSG:3857` = Web Mercator, for basemaps, **never for area/distance computation**.

Pixel size in EPSG:4326 is in *degrees*, and a 0.0001° pixel is ~11 m in latitude but ~11·cos(lat) m in longitude. Any CNN trained on unprojected lat/lon data sees anisotropic pixels that stretch with latitude. Reproject to a metric CRS (UTM zone, or an equal-area CRS for large extents) before tiling.

```python
import numpy as np, rasterio
from rasterio.warp import calculate_default_transform, reproject, Resampling

dst_crs = "EPSG:32633"
with rasterio.open("in.tif") as src:
    transform, width, height = calculate_default_transform(
        src.crs, dst_crs, src.width, src.height, *src.bounds,
        resolution=(10.0, 10.0),          # pin it; otherwise you get an odd derived pixel size
    )
    profile = src.profile.copy()
    profile.update(crs=dst_crs, transform=transform, width=width, height=height)

    with rasterio.open("out.tif", "w", **profile) as dst:
        for i in range(1, src.count + 1):
            reproject(
                source=rasterio.band(src, i),
                destination=rasterio.band(dst, i),
                src_transform=src.transform, src_crs=src.crs, src_nodata=src.nodata,
                dst_transform=transform,     dst_crs=dst_crs, dst_nodata=src.nodata,
                resampling=Resampling.bilinear,
                num_threads=4,
            )
```

**If you omit `src_nodata`/`dst_nodata`, the warp fills the rotated-corner triangles with 0.** Zero is a plausible reflectance value, so nothing errors and your images acquire black wedges that the model happily learns.

### Resampling choice is not cosmetic

| Data | Upsample / same res | Downsample |
|---|---|---|
| Class labels, land-cover, masks | `Resampling.nearest` | `Resampling.mode` (majority) |
| Continuous reflectance, DEM, temperature | `bilinear` (safe) or `cubic` (sharper, can overshoot past valid range) | `average` |
| Binary/probability maps you will threshold | `bilinear` then re-threshold | `average` then re-threshold |
| Categorical you must downsample without inventing classes | — | `mode`, never `average` |

`bilinear` on an integer class raster produces intermediate integers after rounding: classes that appear in your data and in no legend. It never raises. Grep your pipeline for any resampling applied to a label array and confirm it is `nearest` or `mode`.

`cubic`/`lanczos` on physical quantities can produce values outside the valid range (negative reflectance near sharp edges). Clip after.

---

## Affine transforms

`src.transform` is `Affine(a, b, c, d, e, f)` mapping (col, row) → (x, y):
`x = a*col + b*row + c`, `y = d*col + e*row + f`. For a north-up raster `b = d = 0`, `a = +pixel_width`, **`e = -pixel_height` (negative)** because row index grows southward while y grows northward. A positive `e` means a south-up raster — rare, and it will flip your imagery relative to your labels.

`c, f` are the coordinates of the **upper-left corner of the upper-left pixel**, not its centre. GDAL's raw geotransform tuple orders the same six numbers as `(c, a, b, f, d, e)` — do not mix the orderings.

```python
row, col = src.index(x, y)     # world -> array index (floor)
x, y     = src.xy(row, col)    # array index -> world, PIXEL CENTRE by default

from rasterio.transform import rowcol, xy   # same, given a bare Affine
```

Every crop, tile, pad, or resample must produce a new transform:

```python
from rasterio.windows import transform as win_transform
tile_tf = win_transform(Window(c0, r0, w, h), src.transform)

# after a resample by factor k (k>1 = coarser):
coarse_tf = src.transform * rasterio.Affine.scale(k, k)
```

If you save patches as `.npy` and lose the transform, the only recovery is re-deriving it from the tiling code, and any pad or edge-clip you did makes that guesswork. Store `(crs_wkt, transform_tuple, src_path, row_off, col_off)` alongside every patch — a small sidecar `.json` or a parquet index.

---

## Aligning multiple rasters

"Aligned" means **identical CRS, identical transform, identical (height, width)**. Nothing less.

```python
import numpy as np, rasterio

def assert_aligned(a_path, b_path):
    with rasterio.open(a_path) as a, rasterio.open(b_path) as b:
        assert a.crs == b.crs, f"CRS: {a.crs} vs {b.crs}"
        assert (a.height, a.width) == (b.height, b.width), f"shape: {a.shape} vs {b.shape}"
        ta, tb = np.array(a.transform).reshape(3, 3), np.array(b.transform).reshape(3, 3)
        # atol is in CRS units: 1e-6 is nanometres in UTM but ~0.1 m in EPSG:4326 degrees
        assert np.allclose(ta, tb, rtol=0, atol=1e-6), f"transform:\n{a.transform}\n{b.transform}"
```

Run this as a dataset-construction assert, not a notebook check. Matching shape alone proves nothing: two 10980×10980 arrays in different UTM zones have the same shape and plot identically with `imshow`.

To force alignment, reproject onto the reference grid explicitly (do **not** let each file pick its own `calculate_default_transform`):

```python
from rasterio.vrt import WarpedVRT
with rasterio.open("ref.tif") as ref, rasterio.open("other.tif") as other:
    with WarpedVRT(other, crs=ref.crs, transform=ref.transform,
                   width=ref.width, height=ref.height,
                   resampling=Resampling.nearest,       # nearest for labels
                   src_nodata=other.nodata, nodata=other.nodata) as vrt:
        aligned = vrt.read(masked=True)                  # lazy: windowed reads work too
```

`WarpedVRT` is the right tool inside a Dataset: it warps only the window you read, so you never materialize a reprojected copy of a 100 GB archive.

When aligning on the command line, `gdalwarp -tap` snaps the output grid to multiples of the pixel size, which is what makes independently warped rasters land on the same lattice. Without `-tap`, two files warped separately can be offset by a fraction of a pixel. `-tap` is only accepted together with `-tr` (gdalwarp errors otherwise):

```powershell
gdalwarp -t_srs EPSG:32633 -tr 10 10 -tap -r near -srcnodata 255 -dstnodata 255 labels.tif labels_utm.tif
```

---

## NetCDF / HDF5 via xarray

```python
import xarray as xr

ds = xr.open_dataset("t2m.nc")                 # engine auto: netcdf4 / h5netcdf
ds = xr.open_dataset("t2m.nc", chunks={"time": 24, "lat": 512, "lon": 512})  # dask-backed

da = ds["t2m"]                                  # DataArray, dims ('time','lat','lon')
da.sel(time="2020-06-15", method="nearest")     # by COORDINATE VALUE
da.isel(time=0)                                 # by INTEGER POSITION
da.sel(lat=slice(50, 40))                        # slice must follow the coord's stored order!
```

`.sel` with a slice on a **descending** latitude coordinate needs `slice(50, 40)`, not `slice(40, 50)` — the wrong order returns an empty array with no warning. Check `ds.lat[0] > ds.lat[-1]`.

Gotchas that bite:
- **`mask_and_scale=True` is the default.** xarray applies `_FillValue`, `scale_factor`, `add_offset` and returns float with NaN. Good — but the unpacked dtype follows the packing attributes (float32 `scale_factor`/`add_offset` → float32; float64 attrs, or an `add_offset` with no declared dtype → float64), so an int16 variable can quietly become float64 and quadruple in memory. Pass `mask_and_scale=False` only if you will reimplement the unpacking, and know that `_FillValue` is then a raw integer you must compare against yourself.
- `decode_times=True` fails on non-CF calendars; `xr.open_dataset(..., decode_times=False)` then `xr.decode_cf` or `cftime` manually.
- `open_mfdataset(paths, combine="by_coords")` silently misorders files when the time coordinate is missing or undecoded. For a known time stack, be explicit: `xr.open_mfdataset(sorted(paths), combine="nested", concat_dim="time", parallel=True)`.
- **Chunk sizes should be multiples of the on-disk HDF5 chunk shape.** A dask chunk that straddles storage chunks re-reads each storage chunk many times; 10× slowdowns are typical. `chunks={}` adopts the file's own chunking and is a good default.
- **HDF5 is not thread-safe, and xarray serializes reads behind a global lock for the `netcdf4` *and* `h5netcdf` engines alike** — more dask threads buy you almost nothing. Swapping the engine is not the fix; use process-based parallelism (`dask.config.set(scheduler="processes")`, or distributed workers with one thread each), or convert the archive to Zarr, which has no such lock.
- xarray is lazy: nothing is read until `.compute()`, `.values`, or a write. A "fast" pipeline that turns out to do all its work in the last line is the normal experience.

### Zarr

```python
ds = xr.open_zarr("cube.zarr", consolidated=True)      # chunks="auto" = the store's own chunking
ds.to_zarr("out.zarr", mode="w", consolidated=True)

ds.to_zarr("out.zarr", compute=False)                        # write schema + coords once...
part.to_zarr("out.zarr", region={"time": slice(k, k + 1)})   # ...then fill from N workers, lock-free
```

- **Chunks are the unit of decompression.** Pulling one pixel's time series from a `(time, y, x)` cube chunked `(1, 2048, 2048)` decompresses a full spatial plane per timestep. Chunk along the axis you sample: patch training wants `(few_time, 512, 512)`; per-pixel time-series work wants the opposite.
- `consolidated=True` turns thousands of metadata GETs into one — essential on object storage, harmless locally, but only if the store was written that way; appending with a non-consolidating writer leaves a stale `.zmetadata`.
- `region=` writes need the target store, its coordinates, and the array shapes to exist already (the `compute=False` template write above). That is how you fill a cube from many workers without a lock, and it is why Zarr beats NetCDF for a parallel preprocessing pass.
- **Zarr has no CRS concept.** Round-trip the georeference with rioxarray (`ds.rio.write_crs(...)` before `to_zarr`, which stores a `spatial_ref` coordinate) or it is gone.
- Zarr v2 and v3 stores are not interchangeable: `zarr-python` 3.x still reads v2, but a v3 store is unreadable by 2.x. Pin `zarr` in the environment file for anything you share.

### rioxarray bridge

```python
import rioxarray  # registers the .rio accessor; the import looks unused, it is not

da = rioxarray.open_rasterio("scene.tif", masked=True, chunks=True)  # dims ('band','y','x')
da.rio.crs, da.rio.transform(), da.rio.nodata
da = da.rio.write_crs("EPSG:4326")                    # when the file/NetCDF has no CRS
out = da.rio.reproject("EPSG:32633", resampling=Resampling.bilinear)
out = da.rio.reproject_match(ref_da)                  # exact grid match — the alignment workhorse
out.rio.to_raster("out.tif", compress="deflate", tiled=True)
```

`reproject_match` is the xarray answer to `WarpedVRT`: it copies the reference's CRS, transform, and shape. Pass `resampling=Resampling.nearest` when the array is categorical — the default is nearest for rioxarray but it is worth stating explicitly at the call site.

Writing a DataArray whose `y` coordinate is **ascending** produces a vertically flipped GeoTIFF or a positive `e` term. `da.rio.to_raster` handles it, but if you build the array by hand, keep `y` descending.

---

## Vector data: geopandas → label raster

```python
import geopandas as gpd, rasterio
from rasterio.features import rasterize

gdf = gpd.read_file("fields.shp")     # or .geojson, .gpkg
print(gdf.crs)                         # None if the .prj is missing — then you MUST set it

with rasterio.open("ref.tif") as ref:
    gdf = gdf.to_crs(ref.crs)          # STEP 1, always, before anything else
    shapes = ((geom, int(val)) for geom, val in zip(gdf.geometry, gdf["class_id"]))
    labels = rasterize(
        shapes,
        out_shape=(ref.height, ref.width),
        transform=ref.transform,
        fill=255,                       # background / ignore_index, NOT 0
        all_touched=False,
        dtype="uint8",
    )
```

Ordering constraint: **reproject the GeoDataFrame to the raster CRS before rasterizing.** `rasterize` does not know the geometries' CRS; it just applies the transform. Mismatched CRS gives an all-`fill` raster (geometries outside the grid) or, worse, a plausible-looking but wrong burn.

- `all_touched=False` (default) burns only pixels whose **centre** falls inside the polygon. Polygons thinner than a pixel — roads, rivers, small field boundaries — disappear entirely. Set `all_touched=True` for thin features, accepting ~half-pixel dilation.
- **Later shapes overwrite earlier ones** where polygons overlap. Sort by priority explicitly if overlaps matter.
- Use `fill=255` (or another reserved id) rather than 0 so "no polygon here" is distinguishable from class 0, and pass that value as `ignore_index` to your loss.
- Shapefile limits: attribute names truncated to 10 characters, no int64, no proper datetime, one geometry type per file. Prefer GeoPackage (`.gpkg`) or GeoJSON for anything you author.
- Invalid geometries (self-intersections) make `rasterize` raise or drop shapes. `gdf.geometry = gdf.geometry.buffer(0)` or `shapely.make_valid` first; check `gdf.geometry.is_valid.all()`.
- Clipping to a raster footprint: `from rasterio.mask import mask` — a plain `import rasterio` does **not** bind `rasterio.mask`, so `rasterio.mask.mask(...)` is an `AttributeError`. Then `mask(src, shapes, crop=True, filled=True, nodata=src.nodata)` returns `(array, new_transform)`; keep the transform, and pass geometries already in the raster's CRS (`gdf.to_crs(src.crs).geometry`).

---

## Sentinel-2 and Landsat specifics

### Sentinel-2 L2A
| Resolution | Bands |
|---|---|
| 10 m | B02 (blue), B03 (green), B04 (red), B08 (NIR) |
| 20 m | B05, B06, B07, B8A, B11, B12, **SCL** |
| 60 m | B01, B09 (B10 is absent in L2A) |

You cannot `np.stack` these. Resample the 20/60 m bands to the 10 m grid with a decimated read (`bilinear` for reflectance, **`nearest` for SCL**):

```python
with rasterio.open(b11_20m) as src:
    b11 = src.read(1, out_shape=(ref_h, ref_w), resampling=Resampling.bilinear)
with rasterio.open(scl_20m) as src:
    scl = src.read(1, out_shape=(ref_h, ref_w), resampling=Resampling.nearest)
```

**Reflectance conversion — the post-2022 trap.** Historically `reflectance = DN / 10000`. Since Processing Baseline 04.00 (products from 2022-01-25 onward) an offset was added:

```
reflectance = (DN + BOA_ADD_OFFSET) / BOA_QUANTIFICATION_VALUE      # offset = -1000, quant = 10000
```

Read the actual values from the product `MTD_MSIL2A.xml` (`BOA_ADD_OFFSET_VALUES_LIST`) rather than hardcoding. If you mix pre- and post-baseline scenes with the old formula, half your dataset is offset by +0.1 reflectance — a domain shift that looks exactly like a real seasonal/sensor effect and will quietly wreck a foundation-model pretraining run.

Also: `DN == 0` means **no data** in S2 L2A, not zero reflectance. Mask it.

**SCL cloud masking** (band values):

```python
# 0 nodata | 1 saturated/defective | 2 dark area | 3 cloud shadow | 4 vegetation
# 5 not-vegetated | 6 water | 7 unclassified | 8 cloud med prob | 9 cloud high prob
# 10 thin cirrus | 11 snow/ice
BAD = {0, 1, 3, 8, 9, 10}          # add 2 and 11 depending on the task
valid = ~np.isin(scl, list(BAD))
```

SCL is a heuristic classifier, not ground truth: it misses thin cirrus and haze, and over-flags bright soil and snow as cloud. Dilate the cloud mask by 2–5 pixels (`scipy.ndimage.binary_dilation`) to catch fringe contamination, and drop any tile whose valid fraction is below ~0.7 rather than training on mostly-masked patches.

### Landsat 8/9 Collection 2 Level-2

```python
surface_reflectance = DN * 0.0000275 - 0.2      # valid DN range 7273..43636
surface_temperature = DN * 0.00341802 + 149.0   # kelvin
```

Values outside the valid DN range unpack to nonsense (negative reflectance); clip to `[0, 1]` after unpacking and mask `DN == 0`.

`QA_PIXEL` is a bitmask, not a class code:

```python
# bit 0 fill | 1 dilated cloud | 2 cirrus | 3 cloud | 4 cloud shadow | 5 snow | 6 clear | 7 water
bad = (qa & 0b0000_0000_0001_1110) != 0        # dilated cloud|cirrus|cloud|shadow
fill = (qa & 1) != 0
valid = ~(bad | fill)
```

Reading it as a categorical raster (`qa == 21824` etc.) works by accident on a handful of common values and fails everywhere else.

---

## Which tool

| Tool | Use when | Avoid when |
|---|---|---|
| **rasterio** | Single/few GeoTIFFs, windowed reads inside a `Dataset`, tiling, precise transform control, writing COGs | Temporal stacks with named dims; anything larger than memory that you want to express as array math |
| **xarray + rioxarray** | NetCDF/HDF5/Zarr, time or band as a labelled dimension, lazy larger-than-memory math, `reproject_match` alignment | A tight per-patch dataloader path — the coordinate/indexing overhead is significant per call |
| **GDAL CLI** (`gdalwarp`, `gdal_translate`, `gdalbuildvrt`) | One-time bulk conversion, mosaicking, building VRTs, COG creation, `-tap` grid snapping. Fastest, parallelizable with `xargs`/`GNU parallel`, no Python memory ceiling | Anything you need to reason about inside training code |
| **geopandas / shapely** | Vector attributes, spatial joins, CRS transforms of geometries, spatially-disjoint train/test splits | Anything raster |
| **stackstac / odc-stac** | Pulling STAC/cloud archives directly into an xarray cube | Local files you already have |
| **torchgeo** | Image and label archives whose files disagree on CRS and resolution: `RasterDataset` + `IntersectionDataset` unify them on the fly, `RandomGeoSampler` samples in map coordinates | A dataset you already aligned and tiled — every query re-warps through a `WarpedVRT`, and the output grid is inherited from whichever file it indexed first |

Rule of thumb: **preprocess with the GDAL CLI, train with rasterio, analyze with xarray.**

---

## Pre-training checklist

Run this once over the assembled dataset and fail loudly:

1. **CRS**: every image and every label file has a non-`None` CRS, and all are equal. Print the set of unique CRSs — expect exactly one.
2. **Grid**: `assert_aligned(image, label)` for every pair. Not shape-only.
3. **Transform carried**: every saved patch has a stored `(crs, transform)`. Round-trip one patch back to a GeoTIFF and open it in QGIS over the source scene.
4. **Nodata declared**: `src.nodata is not None` for every file, or a documented sentinel per product. Count valid pixels; a valid fraction of exactly 1.0 across a whole satellite archive means your mask is not working.
5. **Stats on valid pixels only**, in float64, and inside the physical range of the product. Store them in the dataset config, not recomputed per run.
6. **Labels**: resampled with `nearest`/`mode` only. `np.unique(labels)` matches the legend exactly, plus the ignore value. Class histogram is non-degenerate (no class at 0 count, none at 99%).
7. **dtype and scale**: reflectance in [0, 1] after unpacking, not [0, 10000] for some scenes and [0, 1] for others. Assert `min >= 0` and `max <= 1.0 + eps` per band across a sample.
8. **Orientation**: plot one image tile and its label tile side by side. A vertical flip from an ascending-`y` write is invisible in every metric until you look.
9. **Split leakage**: split spatially (disjoint tiles/scenes with a buffer of at least the receptive field), never randomly over overlapping patches. Adjacent patches from one scene in train and val is the most common inflated-accuracy bug in remote-sensing papers.

---

## Windows installation

This stack is C libraries (GDAL, PROJ, GEOS) behind Python bindings, and mixing sources produces DLL-load failures and `PROJ: proj_create_from_database: Cannot find proj.db`.

**Best:** conda-forge, one environment, one channel.

```powershell
conda create -n geo -c conda-forge python=3.11 gdal rasterio rioxarray xarray netcdf4 h5netcdf geopandas dask
conda activate geo
```
(Use `mamba`/`micromamba` for the solve; conda's solver on this dependency graph is slow.)

**Also fine:** `uv pip install rasterio geopandas rioxarray` — rasterio (≥1.3) and pyogrio ship Windows wheels with GDAL **bundled inside the wheel**. Fast, no conda.

**Do not mix the two.** conda-forge GDAL plus a pip rasterio wheel means two GDAL copies in one process; the failure is an import-time DLL error or a hard crash mid-read.

- Never `pip install gdal` on Windows and expect a source build to work. There is no reliable path.
- If you have a system GDAL (OSGeo4W/QGIS) on `PATH`, its `GDAL_DATA` and `PROJ_LIB` environment variables will hijack your Python env. Unset them, or set them to the env's own `Library/share/gdal` and `Library/share/proj`.
- Set `PROJ_NETWORK=OFF` on an offline machine; otherwise PROJ stalls trying to fetch datum grids.
- PyTorch DataLoader on Windows uses **spawn**, not fork. Open GDAL datasets inside `__getitem__` or `worker_init_fn`, never in `__init__`, or workers will fail to pickle the handle.
- Paths: use `pathlib.Path` and pass `str(path)`. GDAL accepts forward slashes; raw `\` inside a Python string literal is the usual culprit for "file not found" on a path that exists.
