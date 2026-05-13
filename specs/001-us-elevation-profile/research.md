# Phase 0 Research: US Elevation Profile Module

**Feature**: `001-us-elevation-profile`
**Date**: 2026-05-13
**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

This document records every non-trivial technical decision underpinning the
implementation plan, together with the rationale and the alternatives that were
considered and rejected. All decisions originate from spec FRs/SCs and the four
constitution gates; nothing here invents new requirements.

---

## R-01: Programming language and version

**Decision**: Python 3.11.

**Rationale**: The user prompt specified Python. 3.11 is the most-supported stable
version available in 2026 on developer workstations, CI runners, and downstream
distros; it carries `tomllib` in the standard library (no `tomli` dependency for
parsing `pyproject.toml`), pattern matching, exception groups, and faster startup vs
3.10. All of our required ecosystem libraries (rasterio, pyproj, numpy) have stable
wheels for 3.11.

**Alternatives considered**:

- **Python 3.10**: still common, but loses `tomllib` (forces an extra dep) and the
  performance improvements. No upside.
- **Python 3.12 / 3.13**: viable, but several enterprises still standardize on 3.11.
  We'll declare `python_requires = ">=3.11"`; later versions are supported but not
  required.

---

## R-02: GeoTIFF reader library

**Decision**: `rasterio` ≥ 1.3.

**Rationale**: `rasterio` is the industry-standard Pythonic wrapper around GDAL for
raster IO. It supports **windowed reads** — i.e. reading only a sub-rectangle of a
GeoTIFF without decoding the rest of the file — which is exactly the access pattern
FR-014 demands (lazy per-query tile reads). It exposes the affine transform and
no-data sentinel directly, simplifying the bilinear interpolation step (FR-019). Wheels
are available for Linux x86_64 from PyPI; no system GDAL build required.

**Alternatives considered**:

- **GDAL Python bindings (`osgeo.gdal`) directly**: lower-level; requires system GDAL
  in many environments; rougher API; reinvents what rasterio already provides.
- **`tifffile`**: lightweight pure-Python GeoTIFF reader; lacks first-class support
  for projected coordinate systems and no-data semantics; would force us to
  re-implement geo-referencing.
- **`xarray` + `rioxarray`**: higher-level; pulls in dask-style abstractions we don't
  need; larger dependency footprint; obscures the windowed-read access pattern.

---

## R-03: Geodesic line sampling

**Decision**: `pyproj` ≥ 3.6, using `pyproj.Geod(ellps="WGS84").inv()` for great-circle
distance and `.npts()` / `.line_lengths()` for intermediate points.

**Rationale**: FR-005 requires the geodesic on the WGS84 ellipsoid (not a flat-earth
or rhumb-line approximation). `pyproj` wraps PROJ, the canonical geodesy library used
by virtually every modern GIS tool. Its `Geod` class produces ellipsoidal distance to
sub-millimetre accuracy and gives us evenly spaced intermediate points along the
geodesic, which is exactly what FR-006 sample-spacing needs.

**Alternatives considered**:

- **Hand-rolled Haversine (sphere)**: simple, but uses a spherical earth and would
  introduce ~0.1–0.5% distance error over 200 km — borderline acceptable for distance
  validation (FR-016) but violates FR-005's WGS84-geodesic requirement.
- **`geographiclib`**: Karney's geodesic library, slightly more accurate than PROJ
  for extreme cases (antipodal points). For US-only 1–200 km paths the accuracy
  difference is undetectable, and it adds a second dependency for no practical gain.

---

## R-04: Numerical core

**Decision**: `numpy` ≥ 1.26 for tile arrays, profile arrays, and NaN semantics.

**Rationale**: GeoTIFFs decode to 2D numpy arrays via rasterio. NaN is a first-class
citizen for our no-data representation (FR-009). Bilinear interpolation (FR-019) is a
4-element weighted sum, trivially expressed in numpy. Profile arrays are 1D float64
sequences. No higher-level dataframe abstraction needed.

**Alternatives considered**:

- **`pandas`**: overkill for a 1D float profile; adds a heavy dependency for column
  metadata we already encode as dataclass fields.
- **Pure Python lists**: 10–100× slower for the 1k–10k-sample profile arrays; would
  force us to hand-write NaN handling.

---

## R-05: Sidecar tile-index format

**Decision**: SQLite database via Python stdlib `sqlite3`, written as a single file
at a configurable path (default `<dataset_root>/.us_elevation_profile_index.sqlite`).

**Rationale**: FR-017 requires a persistent index that a fresh process can reload in
seconds (SC-009: < 5 s). FR-018 requires selective refresh of stale entries. SQLite
matches every requirement: durable single-file format, atomic writes within
transactions, indexable bbox queries (an R-tree extension is available in stdlib via
`CREATE VIRTUAL TABLE … USING rtree`), supports concurrent reads, ships with Python's
stdlib (no new dependency to justify against Principle III). For ~1,756 rows the
sidecar will be a few MB.

**Alternatives considered**:

- **JSON sidecar**: simpler to inspect; lacks atomic update semantics (FR-018's
  selective refresh becomes a read-modify-write race); lacks indexed bbox lookup; loads
  whole file into memory on every open.
- **Parquet**: columnar, fast to scan; no random updates without rewriting the file;
  requires an extra dependency (pyarrow); overkill for a few thousand rows.
- **CSV / TSV**: same issues as JSON; even worse for atomic updates.
- **DuckDB embedded**: more powerful query engine; extra dependency; overkill.

---

## R-06: In-memory working-set cache

**Decision**: An LRU cache of decoded numpy tile arrays, sized by total resident
bytes (not by entry count), with a configurable budget defaulting to **1 GB**.

**Rationale**: FR-015 requires bounded resident memory independent of dataset size,
and FR-014 forbids eager loading. A byte-sized LRU is the standard solution: each
tile's decoded array contributes its `nbytes` to the budget; on insertion we evict
least-recently-used tiles until the budget is satisfied. 1 GB fits roughly 10 tiles
at 1 arc-second native (≈ 100 MB each) — enough for any realistic profile across a
few adjacent tiles while leaving generous headroom on a 16 GB laptop. The budget is a
constructor parameter so power-users on bigger machines can raise it.

**Alternatives considered**:

- **`functools.lru_cache`**: counts entries, not bytes — useless when tile sizes
  vary by an order of magnitude (1 arc-sec vs 1/3 arc-sec).
- **mmap the GeoTIFF files**: doesn't decode the compressed/strip data; rasterio
  already memory-maps where the file format allows.
- **`cachetools.LRUCache` with custom getsizeof**: works, but adds a dependency for
  what is ~30 lines of code; Principle III says new deps must clear a higher bar.
  We will write a small in-house byte-bounded LRU.

---

## R-07: Bilinear interpolation strategy

**Decision**: Compute bilinear weights from the (lat, lon) target and the affine
transform of the containing tile; sample the 2×2 neighbourhood with a single numpy
view; propagate no-data strictly (any of the four cells is no-data ⇒ result is NaN,
per FR-019).

**Rationale**: This is the standard formulation. Doing it explicitly (rather than
calling rasterio's `read(..., resampling=Resampling.bilinear)` which operates over
windows) keeps the no-data semantics under our control — rasterio's built-in bilinear
will silently produce values from partially-no-data neighbourhoods, which FR-019
forbids. Cost is trivial: 4 reads + 4 multiplies + 3 adds per sample.

**Alternatives considered**:

- **rasterio's built-in resampling**: violates FR-019's strict no-data propagation.
- **scipy `RegularGridInterpolator`**: heavier dependency; we'd still have to
  post-process to enforce the no-data rule.

---

## R-08: Geodesic sample spacing default (FR-006)

**Decision**: Default sample spacing equals the **native horizontal resolution of the
finest tile the path crosses**, computed in metres at the path's midpoint latitude
(typical 10 m for 1/3 arc-sec NED, ~30 m for 1 arc-sec NED). The caller may override
with either `num_samples` or `spacing_m`.

**Rationale**: Default sampling that matches DEM resolution captures every terrain
feature visible in the source data without oversampling. A 100 km path at 10 m spacing
yields 10,000 samples — within the < 2 s budget (SC-001) for bilinear lookups against
a hot LRU cache. Going finer doesn't gain accuracy (no new source information);
going coarser drops detail.

**Alternatives considered**:

- **Fixed sample count (e.g., 1024)**: ignores DEM resolution; coarse for long paths,
  oversampled for short ones.
- **Caller-must-specify (no default)**: violates FR-006 explicitly.

---

## R-09: Test, coverage, benchmark, property-test stack

**Decision**: `pytest` ≥ 8.0 with `pytest-cov` ≥ 4.1, `pytest-benchmark` ≥ 4.0, and
`hypothesis` ≥ 6.0.

**Rationale**: These are the constitution's required ingredients (Principles II and
IV) realised as concrete tooling. `pytest-cov` enforces ≥ 80% line and ≥ 70% branch
coverage on changed code (`--cov-branch --cov-fail-under=80`). `pytest-benchmark`
runs the SC-001/002/008/009 budgets and auto-fails CI on > 5% regression. `hypothesis`
provides cheap, high-coverage protection on FR-007 (coordinate validation) and FR-016
(distance range), using fuzzed-but-seeded inputs.

**Alternatives considered**:

- **`unittest` (stdlib)**: lacks parametrize, fixtures, plugins for coverage and
  benchmark; would multiply boilerplate.
- **`nose2`**: declining usage; weaker plugin ecosystem.

---

## R-10: Lint, format, type-check tooling

**Decision**: `ruff` ≥ 0.5 for lint + format + import sort + docstring rules;
`mypy` ≥ 1.10 in strict mode on `src/`.

**Rationale**: Principle III requires a single coding style enforced by tooling.
`ruff` collapses what was formerly five tools (flake8, isort, pyupgrade, black,
pydocstyle) into one fast Rust binary with a single config block in `pyproject.toml`.
`mypy --strict` catches type errors before code review and forces every public symbol
to be annotated, supporting Principle I's docstring/annotation requirement.

**Alternatives considered**:

- **black + isort + flake8 + pydocstyle + pyupgrade**: works, but five tools, five
  configs, slower; nothing ruff doesn't provide.
- **pyright**: alternative type checker; equally good; mypy chosen for ubiquity and
  pre-existing community familiarity.

---

## R-11: Logging and error semantics

**Decision**: Stdlib `logging` with module-level loggers named
`us_elevation_profile.<submodule>`; one custom exception hierarchy rooted at
`ElevationError` with concrete subclasses `OutOfCoverageError`, `OutOfRangeError`,
`InvalidCoordinateError`, `StaleIndexError`, `TileReadError`.

**Rationale**: Principle III requires uniform error format and logging structure
across modules. A single hierarchy lets callers catch broad (`except ElevationError`)
or narrow (`except OutOfRangeError`); each subclass corresponds to an FR-defined
error condition so spec→code traceability is trivial. Stdlib `logging` avoids a new
dependency; module-level loggers let the caller control verbosity per submodule.

**Alternatives considered**:

- **`structlog`**: pleasant structured output; adds a runtime dependency for what
  stdlib `logging` already supports via `extra={...}` and a small custom formatter.
- **`loguru`**: monkey-patches stdlib logging; rejected for the same Principle III
  bar on new deps.
- **One generic `ElevationError`**: forces every caller to inspect a message string
  to disambiguate; brittle and unprincipled.

---

## R-12: Concurrency model

**Decision**: v1 ships a **single-threaded** public API. The `ElevationStore` is
explicitly **not** declared thread-safe in its docstring; callers requiring concurrent
access must serialize with their own lock.

**Rationale**: rasterio is generally not safe to call on the same dataset handle from
multiple threads; SQLite is safe with caller serialization but has restrictions on
shared connections. Building a thread-safe surface in v1 adds a lock surface and a
test matrix that the spec does not ask for. Documenting the constraint up front lets
callers do the right thing.

**Alternatives considered**:

- **Per-tile lock + connection pool**: real engineering work, not asked for, can be
  added in a later iteration without breaking the v1 public API.
- **Process-level isolation only**: technically what we ship, but explicit
  documentation prevents misuse.

---

## R-13: Distribution and packaging

**Decision**: PEP 621 `pyproject.toml`; build backend `hatchling`; distribution
name `us-elevation-profile`; import name `us_elevation_profile`; editable install
during development (`pip install -e .[dev]`).

**Rationale**: Modern Python packaging standard. `hatchling` is a minimal,
configuration-light build backend bundled with `hatch`; no setup.py required. The
hyphen-vs-underscore split between distribution and import name is standard
practice and matches Python identifier rules.

**Alternatives considered**:

- **setuptools**: works, but heavier configuration and slower default builds.
- **poetry**: full dependency manager; pulls in its own lock-file workflow that adds
  ceremony without giving us anything the constitution requires.
- **flit**: similar to hatchling; either would work; hatchling chosen for the
  slightly richer plugin ecosystem (e.g., versioning, packaging hooks).

---

## R-14: Sample / fixture data

**Decision**: Tests use small synthetic GeoTIFF tiles (1° × 1° at coarsened
resolution, e.g., 60 × 60 cells) generated at test-collection time by a
`tests/conftest.py` fixture and cached under `tests/data/`. Total committed fixture
size kept under 10 MB.

**Rationale**: Committing real 1-arc-second NED tiles (~100 MB each) into a Git repo
violates Principle III (no sneaking large binaries into the codebase) and bloats the
clone for every contributor. Synthetic tiles let unit and integration tests run
deterministically without external downloads; benchmarks that need *realistic* tile
sizes can be opt-in (driven by an env var pointing at a local NED extract).

**Alternatives considered**:

- **Download real NED tiles in CI**: introduces a network dependency, slows CI,
  fragile to USGS endpoint changes.
- **Use Git LFS**: works but adds a setup step for every clone; not justified by the
  small set of fixtures we need.

---

## Resolved NEEDS-CLARIFICATION list

The plan's Technical Context was filled with concrete values; no entries are marked
`NEEDS CLARIFICATION`. Every dimension (language, deps, storage, tests, target,
project type, perf goals, constraints, scale) is resolved by one of R-01 – R-14
above.
