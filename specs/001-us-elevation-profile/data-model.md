# Data Model: US Elevation Profile Module

**Feature**: `001-us-elevation-profile`
**Date**: 2026-05-13
**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

This document maps every Key Entity from the spec to its concrete in-process shape
(dataclass / typed dict / sqlite table). All fields are testable per Principle II
(every FR cited below has a corresponding contract / integration test in
`tests/contract/` or `tests/integration/`).

---

## E-01: `ElevationStore`

The user-facing object that holds a registered dataset. One instance per registered
dataset root.

| Field | Type | Source | Notes |
|-------|------|--------|-------|
| `dataset_root` | `pathlib.Path` | constructor | Directory the user registered (FR-001); must exist. |
| `index_path` | `pathlib.Path` | constructor or derived | Defaults to `<dataset_root>/.us_elevation_profile_index.sqlite` (R-05). |
| `working_set_bytes` | `int` | constructor; default `1_073_741_824` (1 GiB) | LRU cache budget (FR-015, R-06). Must be > 0. |
| `_index` | `_SqliteIndex` | internal | Lazy-opened SQLite connection wrapper. |
| `_cache` | `_ByteBoundedLRU[TileKey, np.ndarray]` | internal | Decoded-tile cache. |
| `_logger` | `logging.Logger` | internal | `us_elevation_profile.store`. |

**Validation rules**:

- Constructor raises `InvalidDatasetRootError` if `dataset_root` does not exist or is
  not a directory.
- Constructor raises `ValueError` if `working_set_bytes <= 0`.

**Public methods** (full contract in `contracts/api.md`):

- `register_directory(path)` — scan `path` for `*.tif`, classify each as a Tile or
  skip with a diagnostic (FR-001, FR-010), insert/refresh index rows (FR-018).
  Returns a registration summary.
- `coverage()` — bounding region of all registered tiles (FR-002).
- `elevation_at(lat, lon)` — single-point bilinear lookup (FR-003, FR-019).
- `profile_between(transmitter, receiver, *, spacing_m=None, num_samples=None)` —
  geodesic profile (FR-004, FR-005, FR-006, FR-009, FR-012, FR-016).
- `close()` — flush the SQLite index and release file handles.

**Lifecycle**:

- Created → (optionally `register_directory`) → query (any number of times) → `close()`.
- Re-opening with the same `dataset_root` reuses the persistent index (FR-017,
  SC-009). Stale rows detected by `(mtime, size, content_hash)` mismatch are refreshed
  on first access (FR-018).

---

## E-02: `Tile`

A single source GeoTIFF as catalogued in the index.

| Field | Type | Source | Notes |
|-------|------|--------|-------|
| `path` | `pathlib.Path` | filesystem | Absolute path to the `.tif`. Stored relative to `dataset_root` in the sidecar so the index moves with the dataset. |
| `bbox` | `BoundingBox` (dataclass: `min_lat`, `max_lat`, `min_lon`, `max_lon`) | rasterio metadata | Coverage extent in WGS84 decimal degrees. |
| `width_px`, `height_px` | `int` | rasterio metadata | Pixel dimensions. |
| `affine` | `tuple[float, float, float, float, float, float]` | rasterio metadata | (a, b, c, d, e, f) of the GeoTIFF affine transform — maps (col, row) → (lon, lat). |
| `nodata` | `float \| None` | rasterio metadata | Sentinel value used by the source for no-data cells; mapped to NaN in returned arrays (FR-019). |
| `dtype` | `str` | rasterio metadata | Source pixel dtype (e.g., `float32`, `float64`, `int16`); converted to `float64` on read for NaN-capability. |
| `mtime` | `float` | filesystem | Used for staleness detection (FR-018). |
| `size_bytes` | `int` | filesystem | Used for staleness detection (FR-018). |
| `content_hash` | `str` | computed | xxhash64 of the first 64 KiB and the file size; cheap fingerprint sufficient to detect rewrites (FR-018). |

**Validation rules**:

- `min_lat < max_lat`, `min_lon < max_lon`, all in [-90, 90] / [-180, 180].
- `width_px > 0`, `height_px > 0`.
- Files failing rasterio open are skipped during registration with a logged
  diagnostic (FR-010); they do NOT raise.

**SQLite schema** (sidecar `tiles` table):

```sql
CREATE TABLE tiles (
  path           TEXT PRIMARY KEY,        -- relative to dataset_root
  min_lat        REAL NOT NULL,
  max_lat        REAL NOT NULL,
  min_lon        REAL NOT NULL,
  max_lon        REAL NOT NULL,
  width_px       INTEGER NOT NULL,
  height_px      INTEGER NOT NULL,
  affine_a       REAL NOT NULL,
  affine_b       REAL NOT NULL,
  affine_c       REAL NOT NULL,
  affine_d       REAL NOT NULL,
  affine_e       REAL NOT NULL,
  affine_f       REAL NOT NULL,
  nodata         REAL,                    -- nullable
  dtype          TEXT NOT NULL,
  mtime          REAL NOT NULL,
  size_bytes     INTEGER NOT NULL,
  content_hash   TEXT NOT NULL
);
CREATE VIRTUAL TABLE tiles_rtree USING rtree(
  rowid,
  min_lat, max_lat,
  min_lon, max_lon
);  -- indexed bbox lookup for point/path queries
```

---

## E-03: `Endpoint`

A geographic point used to anchor a profile request.

```python
@dataclass(frozen=True, slots=True)
class Endpoint:
    latitude: float   # WGS84 decimal degrees, -90 ≤ lat ≤ 90
    longitude: float  # WGS84 decimal degrees, -180 ≤ lon ≤ 180
    label: Literal["transmitter", "receiver"]
```

**Validation rules** (FR-007, FR-012):

- `-90 <= latitude <= 90` else `InvalidCoordinateError`.
- `-180 <= longitude <= 180` else `InvalidCoordinateError`.
- `label` is one of the two literals; mistyping is a TypeError caught by mypy.

---

## E-04: `ProfileSample`

One observation along the path. Profile-sample arrays are numpy structured arrays
under the hood for speed; the `ProfileSample` dataclass is the per-element view
returned by `__getitem__` for convenience.

```python
@dataclass(frozen=True, slots=True)
class ProfileSample:
    distance_m: float       # great-circle metres from the transmitter (≥ 0)
    latitude: float         # WGS84 decimal degrees
    longitude: float        # WGS84 decimal degrees
    elevation_m: float      # metres above MSL, or NaN for no-data (FR-009, FR-019)
```

**Invariants** (verified by contract tests):

- `distance_m[0] == 0.0` (first sample is the transmitter).
- `distance_m[-1] ≈ total_length_m` within float tolerance (last sample is the receiver).
- `distance_m` is strictly monotonically increasing.
- `latitude` / `longitude` are always finite; only `elevation_m` may be NaN.

---

## E-05: `PathProfile`

The aggregate returned by `profile_between(...)`.

```python
@dataclass(frozen=True, slots=True)
class PathProfile:
    transmitter: Endpoint
    receiver: Endpoint
    total_length_m: float                 # great-circle length, ≥ 1 km, ≤ 200 km (FR-016)
    sample_spacing_m: float               # effective spacing used
    samples: np.ndarray                   # structured array; dtype matches ProfileSample
    no_data_segments: list[tuple[int, int]]   # (start_index, end_index) inclusive ranges of NaN runs (FR-009)
```

**Invariants** (verified by contract tests):

- `len(samples) >= 2` (transmitter and receiver always present).
- `total_length_m` is between 1_000.0 and 200_000.0 inclusive (FR-016).
- `no_data_segments` is sorted and non-overlapping; every `(s, e)` satisfies
  `s <= e` and every sample index in `[s, e]` has `elevation_m == NaN`.
- `samples` accessed as `profile[i]` returns a `ProfileSample` view (FR-009).

---

## E-06: Error hierarchy

```python
class ElevationError(Exception):
    """Base for all module-raised errors."""

class InvalidDatasetRootError(ElevationError):
    """Constructor pointed at a non-existent or non-directory path."""

class InvalidCoordinateError(ElevationError, ValueError):
    """Latitude outside [-90, 90] or longitude outside [-180, 180]."""

class OutOfRangeError(ElevationError, ValueError):
    """Endpoint pair distance outside the 1 km – 200 km supported range (FR-016)."""

class OutOfCoverageError(ElevationError, LookupError):
    """Point/path requested where no registered tile provides data."""

class StaleIndexError(ElevationError):
    """Index sidecar references files that have moved/changed and refresh failed."""

class TileReadError(ElevationError, IOError):
    """Decoding the underlying GeoTIFF failed (corruption, permissions, etc.)."""
```

Every error condition surfaced by the spec maps to one and only one of the above —
this is what makes Principle III's "uniform error format" testable.

---

## Mapping: spec FR-* → entity / field

| FR | Where realised |
|----|----------------|
| FR-001 (register data) | E-01 `ElevationStore.register_directory`, E-02 `Tile` |
| FR-002 (coverage) | E-01 `ElevationStore.coverage()` |
| FR-003 (point query) | E-01 `ElevationStore.elevation_at` |
| FR-004 (profile) | E-01 `ElevationStore.profile_between` → E-05 `PathProfile` |
| FR-005 (geodesic WGS84) | `path.py` (R-03 pyproj) |
| FR-006 (sample spacing) | `path.py` (R-08 default) + `profile_between` kwargs |
| FR-007 (coord validation) | E-03 `Endpoint`, E-06 `InvalidCoordinateError` |
| FR-008 (distinguish out-of-coverage) | E-06 `OutOfCoverageError` + NaN in E-04 |
| FR-009 (no-data shape) | E-04 `ProfileSample.elevation_m = NaN`, E-05 `no_data_segments` |
| FR-010 (tolerate malformed files) | E-01 `register_directory` skip + log diagnostic |
| FR-011 (metres above MSL) | E-04 `elevation_m`, documented unit |
| FR-012 (tx/rx labelling) | E-03 `Endpoint.label`, E-05 `transmitter` / `receiver` |
| FR-013 (1,756-tile scale) | E-01 byte-bounded LRU + lazy tile loading |
| FR-014 (no eager loading) | `tile.py` uses rasterio windowed reads only |
| FR-015 (working-set budget) | E-01 `working_set_bytes`, `cache._ByteBoundedLRU` |
| FR-016 (1–200 km range) | E-05 `total_length_m` invariant, E-06 `OutOfRangeError` |
| FR-017 (persistent index) | E-02 SQLite `tiles` schema, E-01 `index_path` |
| FR-018 (stale detection) | E-02 `mtime`, `size_bytes`, `content_hash` columns |
| FR-019 (bilinear + strict no-data) | `interpolate.py` |

Every FR has a home in the data model. Every entity in the data model is a public
or test-visible surface, supporting Principle II's spec-to-test traceability.
