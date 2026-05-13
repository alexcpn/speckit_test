# Public API Contract: `us_elevation_profile`

**Feature**: `001-us-elevation-profile`
**Date**: 2026-05-13
**Spec**: [../spec.md](../spec.md) | **Plan**: [../plan.md](../plan.md) | **Data model**: [../data-model.md](../data-model.md)

This document is the **frozen** public surface of the library. Every symbol named
here MUST have a corresponding contract test under `tests/contract/` (Principle II);
changes to this file require a coordinated test update in the same PR.

Anything not listed here is internal and may change without notice.

---

## Top-level imports

```python
from us_elevation_profile import (
    ElevationStore,           # E-01
    Endpoint,                 # E-03
    ProfileSample,            # E-04 (view-only)
    PathProfile,              # E-05
    BoundingBox,              # helper
    # Error hierarchy (E-06):
    ElevationError,
    InvalidDatasetRootError,
    InvalidCoordinateError,
    OutOfRangeError,
    OutOfCoverageError,
    StaleIndexError,
    TileReadError,
)
```

Nothing else is re-exported from `__init__.py`. Internal modules (`store`, `tile`,
`cache`, `interpolate`, `path`, `profile`, `errors`) may be imported by tests but
are not part of the contract.

---

## `class ElevationStore`

```python
class ElevationStore:
    def __init__(
        self,
        dataset_root: str | os.PathLike[str],
        *,
        index_path: str | os.PathLike[str] | None = None,
        working_set_bytes: int = 1_073_741_824,  # 1 GiB
    ) -> None: ...
```

### Constructor

**Behaviour**:

- Resolves `dataset_root` to an absolute path; raises `InvalidDatasetRootError` if it
  is not an existing directory.
- If `index_path` is `None`, derives it as
  `<dataset_root>/.us_elevation_profile_index.sqlite`.
- Opens (or creates on first call) the SQLite sidecar. If the sidecar exists it is
  loaded; rows whose `(mtime, size_bytes)` no longer match disk are flagged for
  refresh on first use (FR-018, lazy refresh).
- Raises `ValueError` if `working_set_bytes <= 0`.

**Performance contract**: with an existing valid sidecar, construction completes in
under **5 seconds** for a ~1,756-tile dataset (SC-009). With no sidecar, construction
is cheap (only opens the empty DB); the user is expected to call
`register_directory` next.

---

### `register_directory(path)`

```python
def register_directory(
    self,
    path: str | os.PathLike[str],
    *,
    recurse: bool = True,
) -> RegistrationSummary: ...
```

**Behaviour**:

- Walks `path` (recursively if `recurse=True`) collecting `*.tif` files.
- For each file, attempts to read metadata via rasterio. Failures (malformed,
  unreadable, non-GeoTIFF) are skipped and logged at WARNING; the call continues
  (FR-010).
- For each successful file:
  - Computes `(mtime, size_bytes, content_hash)`.
  - INSERTs a new row, or UPDATEs an existing one whose fingerprint changed.
- DELETE rows whose file path no longer exists on disk (FR-018).
- Returns a `RegistrationSummary`:

  ```python
  @dataclass(frozen=True)
  class RegistrationSummary:
      added: int
      updated: int
      removed: int
      skipped: int
      skipped_paths: list[pathlib.Path]    # paths skipped with reasons logged
      total_after: int                     # rows in index after operation
  ```

**Performance contract**: Registering ~1,756 tiles cold (no prior index) completes in
**< 5 minutes** on a typical developer workstation (SC-008).

**Errors**:

- `InvalidDatasetRootError` if `path` does not exist or is not a directory.
- Individual unreadable files do NOT raise — they appear in `skipped_paths` (FR-010).

---

### `coverage()`

```python
def coverage(self) -> BoundingBox: ...
```

Returns the union bbox over all registered tiles (FR-002). For an empty index,
returns `BoundingBox(min_lat=nan, max_lat=nan, min_lon=nan, max_lon=nan)`.

---

### `elevation_at(lat, lon)`

```python
def elevation_at(self, latitude: float, longitude: float) -> float: ...
```

**Behaviour**:

- Validates `-90 <= latitude <= 90` and `-180 <= longitude <= 180`; otherwise raises
  `InvalidCoordinateError` (FR-007).
- Locates the tile(s) whose bbox contains the point via the R-tree index
  (FR-014: only tiles actually intersected are touched).
- If no tile covers the point, raises `OutOfCoverageError` (FR-008).
- Reads the 2×2 neighbourhood via a rasterio windowed read; applies bilinear
  interpolation (FR-019).
- If any of the four bracketing cells is the tile's no-data sentinel, returns `NaN`
  (FR-019 strict propagation).

**Performance contract**: < **50 ms** for a point in loaded coverage on a typical
developer workstation (SC-002), assuming the relevant tile is hot in the LRU cache;
< 200 ms cold (one decode pass).

**Returns**: elevation in metres above MSL, or `NaN` if the bracketing neighbourhood
includes no-data cells.

---

### `profile_between(transmitter, receiver, *, ...)`

```python
def profile_between(
    self,
    transmitter: Endpoint,
    receiver: Endpoint,
    *,
    num_samples: int | None = None,
    spacing_m: float | None = None,
) -> PathProfile: ...
```

**Behaviour**:

- Validates both endpoints (FR-007). `transmitter.label` must be `"transmitter"` and
  `receiver.label` must be `"receiver"`; otherwise `ValueError` (FR-012).
- Computes the great-circle distance using pyproj on WGS84 (FR-005).
- Validates `1_000.0 <= distance_m <= 200_000.0`; otherwise raises
  `OutOfRangeError` (FR-016). The error's `side` attribute is `"below"` or `"above"`.
- Selects sample spacing:
  - If `num_samples` is given, uses exactly that many samples.
  - Else if `spacing_m` is given, uses that spacing.
  - Else defaults to the native horizontal resolution at the path's midpoint
    (R-08).
- For each sample, computes (lat, lon) via `pyproj.Geod.npts` and elevation via
  `elevation_at`-style bilinear interpolation; samples outside any registered tile or
  in no-data neighbourhoods receive `elevation_m = NaN` (FR-009).
- Builds the `no_data_segments` derived field by scanning for contiguous NaN runs.
- Returns a `PathProfile` (E-05).

**Performance contract**: < **2 seconds** for any path between 1 km and 200 km within
loaded coverage on a typical developer workstation (SC-001).

**Errors**:

- `InvalidCoordinateError` — bad lat/lon (FR-007).
- `OutOfRangeError` — distance not in [1 km, 200 km] (FR-016).
- `ValueError` — wrong endpoint label, contradictory `num_samples` and `spacing_m`
  given together, etc.

`OutOfCoverageError` is **not** raised by `profile_between`: missing coverage along
the path surfaces as NaN samples plus a `no_data_segments` entry (FR-009).

---

### `close()`

```python
def close(self) -> None: ...
def __enter__(self) -> ElevationStore: ...
def __exit__(self, exc_type, exc, tb) -> None: ...   # always calls close()
```

Flushes any pending sidecar writes, closes the SQLite connection, releases rasterio
handles, drops the LRU cache. After `close()`, all query methods raise
`RuntimeError`.

---

## Free functions

The module also exposes two convenience wrappers for callers who want a one-shot
query without retaining a store:

```python
def elevation_at(
    dataset_root: str | os.PathLike[str],
    latitude: float,
    longitude: float,
) -> float: ...

def profile_between(
    dataset_root: str | os.PathLike[str],
    transmitter: Endpoint,
    receiver: Endpoint,
    *,
    num_samples: int | None = None,
    spacing_m: float | None = None,
) -> PathProfile: ...
```

Each opens an `ElevationStore`, dispatches to the corresponding method, then closes
the store. The persistent sidecar is reused across calls (SC-009 < 5 s reopen),
making these wrappers cheap for casual use.

---

## Helper types

```python
@dataclass(frozen=True, slots=True)
class BoundingBox:
    min_lat: float
    max_lat: float
    min_lon: float
    max_lon: float

    def contains(self, lat: float, lon: float) -> bool: ...
```

```python
@dataclass(frozen=True, slots=True)
class RegistrationSummary:
    added: int
    updated: int
    removed: int
    skipped: int
    skipped_paths: list[pathlib.Path]
    total_after: int
```

---

## Compatibility & versioning

- Public surface follows semver. Removing a symbol, narrowing an input domain, or
  widening an output domain are breaking and require a MAJOR bump.
- Adding new optional kwargs with safe defaults, new free functions, new error
  subclasses, or fields on a frozen dataclass (with safe defaults) is MINOR.
- The error hierarchy is part of the contract: catching `ElevationError` or any
  named subclass must keep working across MINOR versions.

---

## Contract-test map

Each subsection above maps to one file under `tests/contract/`:

| Public symbol | Contract test file |
|---------------|--------------------|
| `ElevationStore.__init__` | `test_store_construction.py` |
| `ElevationStore.register_directory` | `test_register_directory.py` |
| `ElevationStore.coverage` | `test_coverage.py` |
| `ElevationStore.elevation_at` | `test_elevation_at.py` |
| `ElevationStore.profile_between` | `test_profile_between.py` |
| `ElevationStore.close` / context manager | `test_store_lifecycle.py` |
| Free `elevation_at` / `profile_between` | `test_free_functions.py` |
| Error hierarchy import & subclass relationships | `test_error_hierarchy.py` |

Constitution Principle II requires every entry above to exist as a failing test
before its implementation lands.
