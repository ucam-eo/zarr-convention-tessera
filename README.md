# Tessera Embedding Convention

- **UUID**: e7f90d5f-019e-4a38-802f-9fa695e26c71
- **Name**: "tessera:"
- **Schema URL**: "<https://raw.githubusercontent.com/ucam-eo/zarr-convention-tessera/refs/tags/v1/schema.json>"
- **Spec URL**: "<https://github.com/ucam-eo/zarr-convention-tessera/blob/v1/README.md>"
- **Scope**: Group
- **Extension Maturity Classification**: Proposal
- **Owner**: @avsm

## Description

This Zarr Convention defines metadata for stores containing quantised geospatial embedding vectors, initially those for the [TESSERA](https://geotessera.org) model (see also the [paper](https://arxiv.org/abs/2506.20380)).  All properties use the `tessera:` namespace prefix and are placed at the root `attributes` level following the [Zarr Conventions Specification](https://github.com/zarr-conventions/zarr-conventions-spec).

A TESSERA store consolidates tiled embedding outputs from a foundation model
into zone-wide Zarr v3 arrays.  Each embedding pixel is stored as an int8
vector together with a float32 per-pixel scale, enabling compact storage with a
well-defined dequantisation contract.  An optional RGBA preview array provides
a visual summary of the embeddings.

Using Int8 quantisation with per-pixel scales reduces embedding storage
significnatly compared to float32, while preserving reconstruction fidelity via
an explicit dequantisation formula.  Ocean areas (marked by NaN scales) are
never written to disk and take advantage of Zarr sparsness.  Grouping tiles by
UTM zones avoids cross-zone projection artefacts and keeps each zone in a
single CRS.

This convention is designed to be composable with other conventions:

- Use with [`proj:`](https://github.com/zarr-conventions/geo-proj) to define the coordinate reference system for each zone
- Use with [`spatial:`](https://github.com/zarr-conventions/spatial) to define the affine transform, bounding box, and spatial dimensions
- Use with [`multiscales`](https://github.com/zarr-conventions/multiscales) for the global preview pyramid

Examples:

- [Zone group (UTM zone 30)](examples/zone-group.json)
- [Year store root](examples/year-store-root.json)
- [Global preview with multiscales](examples/global-preview.json)


## Convention Registration

The convention must be registered in `zarr_conventions`:

```json
{
  "zarr_conventions": [
    {
      "schema_url": "https://raw.githubusercontent.com/ucam-eo/zarr-convention-tessera/refs/tags/v1/schema.json",
      "spec_url": "https://github.com/ucam-eo/zarr-convention-tessera/blob/v1/README.md",
      "uuid": "e7f90d5f-019e-4a38-802f-9fa695e26c71",
      "name": "tessera:",
      "description": "Quantised geospatial embedding vectors with per-pixel dequantisation scales"
    }
  ]
}
```

## Applicable To

This convention can be used with these parts of the Zarr hierarchy:

- [x] Group
- [ ] Array

## Store Layout

A tessera dataset is organised as one Zarr v3 store per year.  Within each store, there is one group per UTM zone that contains data, plus an optional global preview group.

```
{year}.zarr/                          # year store (root group)
    zarr.json                         # tessera: + proj: + spatial: metadata
    utm{zone:02d}/                    # zone group (one per populated UTM zone)
        zarr.json                     # tessera: + proj: + spatial: metadata
        embeddings                    # int8    (H, W, B)    shards=(256,256,B)
        scales                        # float32 (H, W)       shards=(256,256)
        rgb                           # uint8   (H, W, 4)    shards=(256,256,4)  [optional]
        easting                       # float64 (W,)         coordinate array
        northing                      # float64 (H,)         coordinate array
        band                          # int32   (B,)         coordinate array
    global_rgb/                       # global EPSG:4326 preview [optional]
        zarr.json                     # multiscales + proj: + spatial: metadata
        0/rgb                         # uint8 (H0, W0, 4)    level 0 (finest)
        1/rgb                         # uint8 (H1, W1, 4)    level 1
        ...
```

### Year Store Root Group

The root group identifies the store as a tessera dataset and declares the year.
It also registers the conventions used by its children.

### Zone Groups

Each zone group contains the core embedding data for all tiles within a single
UTM zone.  The zone grid is a rectangular region in the zone's native UTM
projection, snapped to shard boundaries (multiples of 256 pixels = 2.56 km).

For `tessera:dataset_version` = `"v1"`, the pixel size is fixed at **10 m** (matching Sentinel-2 resolution).  This is encoded in the `spatial:transform` affine coefficients and is not stored as a separate tessera attribute.

### Global Preview Group (optional)

The global preview is an RGBA image pyramid in EPSG:4326 covering the full
Earth extent.  It uses the `multiscales` convention to describe the pyramid
levels.  Each level is derived from the zone-level RGB previews by reprojecting
from UTM to geographic coordinates and compositing overlapping zones.

## Properties

All properties use the `tessera:` namespace prefix and are placed at the root
`attributes` level.

### Year Store Root Properties

| Property                      | Type     | Description                                              | Required | Reference                                                     |
| ----------------------------- | -------- | -------------------------------------------------------- | -------- | ------------------------------------------------------------- |
| **tessera:dataset_version**   | `string` | Version of the tessera model schema (e.g., `"v1"`)       | Yes      | [tessera:dataset_version](#tesseradataset_version)           |
| **tessera:year**              | `integer`| Year this store covers                                   | Yes      | [tessera:year](#tesserayear)                                 |

### Zone Group Properties

| Property                      | Type      | Description                                                          | Required | Reference                                                     |
| ----------------------------- | --------- | -------------------------------------------------------------------- | -------- | ------------------------------------------------------------- |
| **tessera:dataset_version**   | `string`  | Version of the tessera model schema                                   | Yes      | [tessera:dataset_version](#tesseradataset_version)           |
| **tessera:year**              | `integer` | Year of the data in this zone                                        | Yes      | [tessera:year](#tesserayear)                                 |
| **tessera:utm_zone**          | `integer` | UTM zone number (1-60)                                               | Yes      | [tessera:utm_zone](#tesserautm_zone)                         |
| **tessera:n_bands**           | `integer` | Number of embedding bands                                            | Yes      | [tessera:n_bands](#tesseran_bands)                           |
| **tessera:n_tiles**           | `integer` | Number of source tiles merged into this zone                         | Yes      | [tessera:n_tiles](#tesseran_tiles)                           |
| **tessera:model_version**     | `string`  | Version of the embedding model that produced the vectors             | Yes      | [tessera:model_version](#tesseramodel_version)               |
| **tessera:build_version**     | `string`  | Version of the software that built this store                        | No       | [tessera:build_version](#tesserabuild_version)               |
| **tessera:has_rgb_preview**   | `boolean` | Whether the zone group contains an RGB preview array                 | No       | [tessera:has_rgb_preview](#tesserahas_rgb_preview)           |

### Field Details

Additional properties are allowed.

#### tessera:dataset_version

Version of the tessera data schema

- **Type**: `string`
- **Required**: Yes
- **Example**: `"v1"`

Identifies the version of the tessera store layout and metadata contract.
Readers SHOULD check this value before attempting to interpret the store.
Changes to the array layout, dequantisation formula, or required properties
constitute a new dataset version. 

#### tessera:year

Year of the data

- **Type**: `integer`
- **Required**: Yes
- **Example**: `2024`

The calendar year for which the embeddings were computed.  Each year store contains data for exactly one year.

#### tessera:utm_zone

UTM zone number

- **Type**: `integer`
- **Required**: Yes (on zone groups)
- **Minimum**: 1
- **Maximum**: 60
- **Example**: `30`

The Universal Transverse Mercator zone number.  The corresponding EPSG code is derived as `EPSG:{32600 + zone}` for northern-hemisphere zones or `EPSG:{32700 + zone}` for southern-hemisphere zones.  The zone group name MUST match `utm{zone:02d}` (e.g., `utm30`).

#### tessera:n_bands

Number of embedding bands

- **Type**: `integer`
- **Required**: Yes (on zone groups)
- **Minimum**: 1
- **Example**: `128`

The number of bands (channels) in the embedding vectors.  This MUST match the size of the last dimension of the `embeddings` array and the length of the `band` coordinate array.

#### tessera:n_tiles

Number of source tiles

- **Type**: `integer`
- **Required**: Yes (on zone groups)
- **Minimum**: 0
- **Example**: `240`

The number of source tiles that were merged into this zone group during the build process.  Useful for provenance tracking and quality assessment.

#### tessera:model_version

Embedding model version

- **Type**: `string`
- **Required**: Yes (on zone groups)
- **Example**: `"1.0"`

Identifies the version of the foundation model whose embeddings are stored.  Different model versions may produce embeddings with different dimensionality, magnitude, or semantic meaning, so readers MUST check this value before comparing embeddings across stores.  Stores built from different model versions SHOULD NOT be mixed without explicit re-embedding or alignment.

#### tessera:build_version

Build software version

- **Type**: `string`
- **Required**: No
- **Example**: `"geotessera-0.14.0"`

The name and version of the software that produced this store.  Useful for debugging and provenance.

#### tessera:has_rgb_preview

RGB preview availability

- **Type**: `boolean`
- **Required**: No
- **Default**: `false`

Indicates whether the zone group contains a populated `rgb` array.  The RGB preview is optional because it can be computed after the embeddings are written.  Readers SHOULD check this flag before attempting to read the `rgb` array.

## Dequantisation

The tessera convention uses per-pixel scale quantisation.  Clients reconstruct float32 embedding vectors from the `embeddings` (int8) and `scales` (float32) arrays using the following formula:

```
embedding_float32[y, x, b] = embeddings[y, x, b] * scales[y, x]
```

Where:
- `embeddings` is the int8 quantised embedding array with shape (H, W, B)
- `scales` is the float32 per-pixel scale array with shape (H, W)
- The same scale value is applied to all B bands at a given pixel
- Where `scales[y, x]` is NaN, the pixel has no valid data (water or no coverage); clients MUST NOT attempt dequantisation for these pixels
- Where `scales[y, x]` is 0.0, all bands at that pixel are zero in the original float32 representation

This formula is fixed by the convention and is NOT recorded in store metadata.  Clients that declare support for the tessera convention MUST implement this dequantisation.

**Python example:**

```python
import numpy as np
import zarr

zone = zarr.open_group("2024.zarr/utm30", mode="r")
embeddings = zone["embeddings"][:]   # int8  (H, W, B)
scales = zone["scales"][:]           # float32 (H, W)

# Dequantise to float32
float_embeddings = embeddings.astype(np.float32) * scales[:, :, np.newaxis]

# No-data mask
valid = ~np.isnan(scales)
```

## Array Specifications

Zone groups MUST contain the following arrays:

### embeddings

Quantised embedding vectors.

| Property       | Value                                        |
| -------------- | -------------------------------------------- |
| **Shape**      | `(H, W, B)` where B = `tessera:n_bands`     |
| **Dtype**      | `int8`                                       |
| **Fill value** | `0`                                          |
| **Dimensions** | `["northing", "easting", "band"]`            |
| **Chunks**     | `(C, C, B)` (inner chunks within shards)     |
| **Shards**     | `(S, S, B)` (shard size in pixels)           |
| **Compression**| Blosc (zstd, clevel 3) recommended           |

The fill value of 0 means that unwritten (sparse) regions produce zero vectors, which combined with a NaN scale unambiguously indicates no data.

### scales

Per-pixel dequantisation scales.

| Property       | Value                                        |
| -------------- | -------------------------------------------- |
| **Shape**      | `(H, W)`                                     |
| **Dtype**      | `float32`                                    |
| **Fill value** | `NaN`                                        |
| **Dimensions** | `["northing", "easting"]`                    |
| **Chunks**     | `(C, C)` (inner chunks within shards)        |
| **Shards**     | `(S, S)` (shard size in pixels)              |
| **Compression**| Blosc (zstd, clevel 3) recommended           |

The fill value of NaN means that unwritten (sparse) regions are automatically no-data.  NaN values also indicate water pixels (as determined by a landmask) or areas with no satellite coverage.

### rgb (optional)

RGBA preview image for visualisation.

| Property       | Value                                        |
| -------------- | -------------------------------------------- |
| **Shape**      | `(H, W, 4)`                                 |
| **Dtype**      | `uint8`                                      |
| **Fill value** | `0`                                          |
| **Dimensions** | `["northing", "easting", "band"]`            |
| **Chunks**     | `(C, C, 4)` (inner chunks within shards)     |
| **Shards**     | `(S, S, 4)` (shard size in pixels)           |
| **Compression**| Blosc (zstd, clevel 3) recommended           |

Channels 0-2 are R, G, B.  Channel 3 is alpha: 255 for valid data, 0 for no-data.  This array is only present when `tessera:has_rgb_preview` is `true`.

The RGB preview is an opaque visualisation aid.  The method used to derive RGB values from embeddings (e.g., band selection, PCA, learned projection) is an implementation detail and is not specified by this convention.  Clients MUST NOT rely on a particular relationship between the `rgb` and `embeddings` arrays.

### Coordinate Arrays

Zone groups MUST also contain the following 1-D coordinate arrays:

| Array        | Shape  | Dtype     | Dimension   | Description                               |
| ------------ | ------ | --------- | ----------- | ----------------------------------------- |
| **easting**  | `(W,)` | `float64` | `"easting"` | UTM easting at pixel centres              |
| **northing** | `(H,)` | `float64` | `"northing"`| UTM northing at pixel centres (descending)|
| **band**     | `(B,)` | `int32`   | `"band"`    | Band indices `[0, 1, ..., B-1]`           |

Coordinate arrays are uncompressed and unsharded.

The `easting` array contains UTM easting values at the centre of each pixel column, computed as `origin_easting + (i + 0.5) * pixel_size` for column index `i`.

The `northing` array contains UTM northing values at the centre of each pixel row, computed as `origin_northing - (j + 0.5) * pixel_size` for row index `j`.  Values are in descending order (north to south).

## Sharding and Chunking

The convention recommends (but does not require) the following sharding parameters:

| Parameter             | Recommended Value | Rationale                                                          |
| --------------------- | ----------------- | ------------------------------------------------------------------ |
| **Shard size (S)**    | 256 pixels        | 2.56 km at the v1 pixel size of 10 m; keeps file count manageable  |
| **Inner chunk (C)**   | 4 pixels          | Enables O(2 KB) single-pixel HTTP range requests for embeddings    |
| **Zarr format**       | 3                 | Required for sharding support                                       |

The zone grid extent is snapped to shard boundaries so that all shards are complete (no partial shards at edges).  Pixels outside the actual data coverage within the grid are sparse (never written), so they consume no storage.

## Composition with Other Conventions

### With proj: and spatial:

Zone groups MUST also declare the `proj:` and `spatial:` conventions with appropriate properties:

- `proj:code`: The EPSG code for the zone's UTM projection (e.g., `"EPSG:32630"`)
- `proj:wkt2`: Full WKT2 CRS definition (recommended)
- `spatial:dimensions`: `["northing", "easting"]`
- `spatial:transform`: 6-element affine `[pixel_size, 0, origin_easting, 0, -pixel_size, origin_northing]`
- `spatial:shape`: `[H, W]`
- `spatial:bbox`: `[easting_min, northing_min, easting_max, northing_max]`
- `spatial:registration`: `"pixel"`

### With multiscales (global preview)

The optional `global_rgb` group uses the `multiscales` convention to describe a power-of-2 pyramid in EPSG:4326.  Each level contains an `rgb` array (uint8, RGBA) and a `band` coordinate array.  The multiscale layout items include `spatial:shape` and `spatial:transform` for each level.

The global preview covers the full Earth extent (-180 to 180 longitude, -90 to 90 latitude).  Level 0 has a resolution of approximately 0.0001 degrees (~10 m at the equator).  Each subsequent level is derived by 2x2 mean reduction.

## UTM Zone Naming and CRS

Zone groups are named `utm{NN}` where `NN` is the two-digit zero-padded UTM zone number (01-60).  The CRS is always a UTM projection:

- **Northern hemisphere zones**: `EPSG:{32600 + zone}` (e.g., zone 30 = EPSG:32630)
- **Southern hemisphere zones**: `EPSG:{32700 + zone}` (e.g., zone 30 south = EPSG:32730)

Southern hemisphere zones use UTM's standard false northing of 10,000,000 m.  The `proj:code` and `proj:wkt2` on the zone group are authoritative for determining the exact CRS.

## Examples

### Zone Group (UTM Zone 30, 2024)

```json
{
  "zarr_format": 3,
  "node_type": "group",
  "attributes": {
    "zarr_conventions": [
      {
        "schema_url": "https://raw.githubusercontent.com/ucam-eo/zarr-convention-tessera/refs/tags/v1/schema.json",
        "spec_url": "https://github.com/ucam-eo/zarr-convention-tessera/blob/v1/README.md",
        "uuid": "e7f90d5f-019e-4a38-802f-9fa695e26c71",
        "name": "tessera:",
        "description": "Quantised geospatial embedding vectors with per-pixel dequantisation scales"
      },
      {
        "schema_url": "https://raw.githubusercontent.com/zarr-conventions/geo-proj/refs/tags/v1/schema.json",
        "spec_url": "https://github.com/zarr-conventions/geo-proj/blob/v1/README.md",
        "uuid": "f17cb550-5864-4468-aeb7-f3180cfb622f",
        "name": "proj:",
        "description": "Coordinate reference system information for geospatial data"
      },
      {
        "schema_url": "https://raw.githubusercontent.com/zarr-conventions/spatial/refs/tags/v1/schema.json",
        "spec_url": "https://github.com/zarr-conventions/spatial/blob/v1/README.md",
        "uuid": "689b58e2-cf7b-45e0-9fff-9cfc0883d6b4",
        "name": "spatial:",
        "description": "Spatial coordinate information"
      }
    ],
    "tessera:dataset_version": "v1",
    "tessera:year": 2024,
    "tessera:utm_zone": 30,
    "tessera:n_bands": 128,
    "tessera:n_tiles": 240,
    "tessera:model_version": "1.0",
    "tessera:build_version": "0.14.0",
    "tessera:has_rgb_preview": true,
    "proj:code": "EPSG:32630",
    "proj:wkt2": "PROJCRS[\"WGS 84 / UTM zone 30N\", ...]",
    "spatial:dimensions": ["northing", "easting"],
    "spatial:transform": [10.0, 0.0, 500000.0, 0.0, -10.0, 6200000.0],
    "spatial:shape": [5000, 6000],
    "spatial:bbox": [500000.0, 6150000.0, 560000.0, 6200000.0],
    "spatial:registration": "pixel"
  }
}
```

### Year Store Root

```json
{
  "zarr_format": 3,
  "node_type": "group",
  "attributes": {
    "zarr_conventions": [
      {
        "schema_url": "https://raw.githubusercontent.com/ucam-eo/zarr-convention-tessera/refs/tags/v1/schema.json",
        "spec_url": "https://github.com/ucam-eo/zarr-convention-tessera/blob/v1/README.md",
        "uuid": "e7f90d5f-019e-4a38-802f-9fa695e26c71",
        "name": "tessera:",
        "description": "Quantised geospatial embedding vectors with per-pixel dequantisation scales"
      },
      {
        "schema_url": "https://raw.githubusercontent.com/zarr-conventions/geo-proj/refs/tags/v1/schema.json",
        "spec_url": "https://github.com/zarr-conventions/geo-proj/blob/v1/README.md",
        "uuid": "f17cb550-5864-4468-aeb7-f3180cfb622f",
        "name": "proj:",
        "description": "Coordinate reference system information for geospatial data"
      },
      {
        "schema_url": "https://raw.githubusercontent.com/zarr-conventions/spatial/refs/tags/v1/schema.json",
        "spec_url": "https://github.com/zarr-conventions/spatial/blob/v1/README.md",
        "uuid": "689b58e2-cf7b-45e0-9fff-9cfc0883d6b4",
        "name": "spatial:",
        "description": "Spatial coordinate information"
      }
    ],
    "tessera:dataset_version": "v1",
    "tessera:year": 2024
  }
}
```

### Global Preview with Multiscales

```json
{
  "zarr_format": 3,
  "node_type": "group",
  "attributes": {
    "zarr_conventions": [
      {
        "schema_url": "https://raw.githubusercontent.com/zarr-conventions/multiscales/refs/tags/v1/schema.json",
        "spec_url": "https://github.com/zarr-conventions/multiscales/blob/v1/README.md",
        "uuid": "d35379db-88df-4056-af3a-620245f8e347",
        "name": "multiscales",
        "description": "Multiscale layout of zarr datasets"
      },
      {
        "schema_url": "https://raw.githubusercontent.com/zarr-conventions/geo-proj/refs/tags/v1/schema.json",
        "spec_url": "https://github.com/zarr-conventions/geo-proj/blob/v1/README.md",
        "uuid": "f17cb550-5864-4468-aeb7-f3180cfb622f",
        "name": "proj:",
        "description": "Coordinate reference system information for geospatial data"
      },
      {
        "schema_url": "https://raw.githubusercontent.com/zarr-conventions/spatial/refs/tags/v1/schema.json",
        "spec_url": "https://github.com/zarr-conventions/spatial/blob/v1/README.md",
        "uuid": "689b58e2-cf7b-45e0-9fff-9cfc0883d6b4",
        "name": "spatial:",
        "description": "Spatial coordinate information"
      }
    ],
    "multiscales": {
      "layout": [
        {
          "asset": "0",
          "transform": {
            "scale": [1.0, 1.0],
            "translation": [0.0, 0.0]
          },
          "spatial:shape": [1800000, 3600000],
          "spatial:transform": [0.0001, 0.0, -180.0, 0.0, -0.0001, 90.0]
        },
        {
          "asset": "1",
          "derived_from": "0",
          "transform": {
            "scale": [2.0, 2.0],
            "translation": [0.0, 0.0]
          },
          "resampling_method": "mean",
          "spatial:shape": [900000, 1800000],
          "spatial:transform": [0.0002, 0.0, -180.0, 0.0, -0.0002, 90.0]
        },
        {
          "asset": "2",
          "derived_from": "1",
          "transform": {
            "scale": [2.0, 2.0],
            "translation": [0.0, 0.0]
          },
          "resampling_method": "mean",
          "spatial:shape": [450000, 900000],
          "spatial:transform": [0.0004, 0.0, -180.0, 0.0, -0.0004, 90.0]
        }
      ],
      "resampling_method": "mean"
    },
    "proj:code": "EPSG:4326",
    "spatial:dimensions": ["lat", "lon"],
    "spatial:bbox": [-180.0, -90.0, 180.0, 90.0]
  }
}
```

## FAQ

### Why use int8 quantisation with per-pixel scales?

Foundation model embeddings are typically float32 vectors.  Storing them
directly at 10m global resolution would require enormous storage.  Int8
quantisation with a per-pixel scale achieves approximately 4x compression while
preserving the relative relationships between embedding dimensions at each
pixel.  The per-pixel scale (rather than a global or per-band scale) handles
the fact that different land cover types produce embeddings with different
magnitudes.

The scale is stored as a separate array rather than interleaved with the
embeddings so that:
- No-data can be indicated by NaN in the scales array (int8 has no NaN)
- The scales array can be read independently for quick coverage checks
- The embeddings array has a uniform dtype throughout

### Why NaN for no-data instead of a sentinel value?

Zarr v3 sparse arrays use the fill value to detect unwritten chunks.  With a
NaN fill value, unwritten pixels are automatically no-data without consuming
storage and unlike sentinel values (e.g., 0 or -9999), NaN cannot be confused
with a valid scale.  Water pixels identified by an external landmask are set to
NaN during the build, ensuring they are treated as no-data regardless of
whether the satellite captured an image there.

### Why organise by UTM zone?

UTM zones provide uniform-metre coordinate systems that avoid the distortion of geographic (lat/lon) coordinates.  Organising by zone means:
- Each zone group is in a single CRS with no projection boundaries within the data
- Pixel size in metres is constant throughout the zone
- Standard geospatial tools can read each zone without reprojection
- The zone structure maps naturally to how satellite imagery is processed and distributed

The trade-off is that global queries must read from multiple zones and reproject, which is why the optional `global_rgb` preview exists.

### Why one store per year?

Temporal organisation by year matches the common pattern of annual satellite composites.  Each year store is independent, so:
- Years can be built and updated independently
- Storage can be added incrementally as new years become available
- Old years can be replaced without affecting others.

However, in the future we may create [Virtual Zarr stores](https://virtualizarr.readthedocs.io/en/stable/) that do expose years as
a dimension.

### How does the global preview relate to the zone data?

The `global_rgb` group is derived from the zone-level RGB previews by:
1. Reprojecting each zone's RGB array from its native UTM CRS to EPSG:4326
2. Compositing overlapping zones (where zones overlap, the best-quality data is preferred)
3. Building a power-of-2 pyramid using 2x2 mean reduction

The global preview does NOT contain embedding data and is purely visual.  For
actual embedding values, readers must access the appropriate zone group and
work in the zone's native UTM projection.

### How does tessera: compose with proj: and spatial:?

The tessera convention is purely additive.  It defines the embedding-specific properties (`tessera:n_bands`, `tessera:model_version`, etc.) and the dequantisation formula, while relying on `proj:` for CRS information and `spatial:` for coordinate transforms.  A reader that does not understand `tessera:` can still interpret the spatial metadata using only `proj:` and `spatial:`.  A reader that does understand `tessera:` can reconstruct float32 embeddings using the dequantisation formula defined by this convention.

### Can the convention support other quantisation methods?

Future versions of the tessera convention MAY define alternative dequantisation
formulas.  Readers MUST check `tessera:dataset_version` to determine which
formula applies.  Version `"v1"` always uses per-pixel-scale quantisation as
described in the [Dequantisation](#dequantisation) section.

## Acknowledgements

The template is based on the [STAC extensions template](https://github.com/stac-extensions/template/blob/main/README.md) and follows the structure established by the [spatial](https://github.com/zarr-conventions/spatial), [geo-proj](https://github.com/zarr-conventions/geo-proj), and [multiscales](https://github.com/zarr-conventions/multiscales) conventions.
We thank Deepak Cherian for much useful feedback on this spec, as well as prototypes from Mark Elvers that helped inform this convention.
