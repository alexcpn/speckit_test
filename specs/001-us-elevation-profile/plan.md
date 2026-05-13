# Implementation Plan: US Elevation Profile Module

**Branch**: `main` (originally `001-us-elevation-profile`; feature branch was collapsed onto main after the clarify step — work continues on main)
**Date**: 2026-05-13
**Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-us-elevation-profile/spec.md`

**Note**: This plan operationalises the spec under the Speckit Test Constitution v1.0.0 (Code Quality, TDD, Consistency, Performance).

## Summary

Deliver an importable Python library that registers USGS 3DEP GeoTIFF elevation tiles
(target dataset: ~1,756 files / ~23 billion 64-bit samples covering CONUS + Hawaii +
Puerto Rico), persists a small on-disk metadata index, and serves two public operations:
(a) bilinearly interpolated elevation at a single (lat, lon), and (b) the great-circle
elevation profile between a transmitter–receiver pair separated by 1–200 km. Resident
memory is bounded by a configurable LRU working-set budget so that a 180-GB raw dataset
never needs to be held in RAM. No-data samples surface as NaN-in-place with a derived
list of contiguous no-data segments.

Technical approach: a thin layered library — `store` (registration + sidecar index) →
`tile` (lazy GeoTIFF reads via rasterio windows) → `cache` (LRU on decoded tile arrays)
→ `interpolate` (bilinear) → `path` (geodesic sampling via pyproj) → `profile`
(PathProfile / ProfileSample dataclasses) → `api` (the public surface). Tests are
pytest-driven (Principle II: tests committed before implementation, ≥80% line / ≥70%
branch coverage gates); benchmarks are pytest-benchmark suites pinned to every SC-* in
the spec (Principle IV: every hot path carries a passing benchmark).

## Technical Context

**Language/Version**: Python 3.11 (long-supported, supports `match` / `ExceptionGroup`,
native `tomllib`; widely available on developer workstations and CI; later 3.12+ allowed
but not required).

**Primary Dependencies**:

- `rasterio` ≥ 1.3 — GeoTIFF reads (windowed reads avoid loading whole tiles; this is
  the bedrock of FR-014 lazy tile access). Wraps GDAL but exposes a Pythonic surface;
  industry default for raster IO in Python.
- `numpy` ≥ 1.26 — tile and profile arrays; NaN semantics for no-data (FR-009).
- `pyproj` ≥ 3.6 — WGS84 geodesic line calculations (FR-005, FR-016 great-circle
  distance for range validation).
- `pytest` ≥ 8.0 — test runner (Constitution Principle II).
- `pytest-benchmark` ≥ 4.0 — automated benchmarks for SC-001/002/008/009 (Principle IV).
- `pytest-cov` ≥ 4.1 — coverage gate enforcement (Principle II).
- `hypothesis` ≥ 6.0 — property-based testing for FR-007 (coord validation) and FR-016
  (range validation); cheap, high-coverage protection against numeric edge cases.

Tooling-only (dev deps, no runtime impact):

- `ruff` ≥ 0.5 — lint, format, import-sort, docstring checks (Principle I single-tool
  style enforcement).
- `mypy` ≥ 1.10 — type checking, strict on the public API surface (Principle I static
  analysis).

**Storage**:

- Source: filesystem (user-supplied directory of GeoTIFF tiles, per FR-001).
- Sidecar index: SQLite via stdlib `sqlite3` — single-file `.us_elevation_profile_index.sqlite`
  written alongside the registered dataset root (or to a user-specified location).
  Stores one row per tile: absolute path, bounding box (min/max lat/lon), resolution,
  pixel dimensions, no-data sentinel, mtime, content hash. Stdlib-only (no new
  dependency); atomic writes; supports point-in-polygon queries via straightforward
  bbox filters; resilient across processes (FR-017, FR-018).
- Working-set cache: in-memory LRU of decoded numpy arrays for hot tiles
  (FR-015 working-set budget, default 1 GB, configurable at store construction).

**Testing**: pytest + pytest-cov + pytest-benchmark + hypothesis. Tests laid out as
`tests/unit/`, `tests/integration/`, `tests/contract/`, `tests/benchmarks/`. Sample
GeoTIFF fixtures kept small (a handful of synthetic 1°×1° tiles, total < 10 MB) and
committed under `tests/data/`.

**Target Platform**: Linux x86_64 (developer workstation + CI). The library is
OS-portable in principle (pure Python + portable C deps for rasterio/numpy/pyproj) but
v1 is verified on Linux only.

**Project Type**: Single Python library, installable from source with `pip install -e .`
and distributable as a wheel via `pyproject.toml` (PEP 621 metadata). No CLI, no
service, no UI in v1.

**Performance Goals** (each mirrors a spec SC-* and becomes a pytest-benchmark assertion):

| Operation | Budget | Source |
|-----------|--------|--------|
| Single-point query (within coverage) | < 50 ms | SC-002 |
| Profile (1 km – 200 km, fully covered) | < 2 s | SC-001 |
| Registration of ~1,756 NED files (cold) | < 5 min | SC-008 |
| Restart with existing index (no changes) | < 5 s | SC-009 |
| Working-set memory (steady-state, full dataset) | ≤ configured budget; default 1 GB | SC-007 |

**Constraints**:

- No part of the API may load all tiles eagerly (FR-014; the LRU cache plus rasterio
  windowed reads are how this is enforced).
- Tile-index sidecar < 100 MB for the full target dataset (estimated < 5 MB realistic).
- v1 is single-threaded from the API's perspective: the public `ElevationStore` is
  **not** declared thread-safe. Callers requiring concurrent access serialize through a
  lock. Rationale: keeps lock surface out of v1; rasterio + SQLite are both safe with
  caller serialization. (Recorded under Complexity Tracking only if a reviewer disputes
  this scope choice.)
- Public errors are a small, typed hierarchy rooted at `ElevationError`
  (Principle III uniform error format).
- All public symbols carry docstrings stating purpose, inputs, outputs, error
  conditions (Principle I).

**Scale/Scope**:

- Target dataset: ~1,756 GeoTIFF tiles, ~23 billion 64-bit samples (~180 GB raw;
  compressed source files are typically much smaller).
- Sidecar index: 1 row per tile; ~30–50 columns of metadata → ~few MB total.
- Spec surface: 19 functional requirements, 9 success criteria, 4 key entities,
  3 user stories.
- Estimated library footprint at completion: ~1,500–2,000 LOC across ~8 modules plus
  a similarly sized test suite. (Principle I complexity limits apply per function,
  not aggregate.)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

The Speckit Test Constitution v1.0.0 defines four gates. Each is addressed concretely:

### Gate 1 — Code Quality (NON-NEGOTIABLE)

| Constitution requirement | Plan compliance |
|--------------------------|-----------------|
| Lint, format, static analysis pass with zero warnings | `ruff check` + `ruff format --check` + `mypy --strict` on `src/` run in CI and as pre-commit hooks |
| Cyclomatic complexity ≤ 10 per function | Enforced by `ruff` rule `C90` (`max-complexity = 10`); violations require PR-recorded justification per the constitution |
| Public docstrings on all symbols | `ruff` rules `D101–D107`, `D200`, `D205`, `D400` enabled; mypy `--strict` rejects untyped public exports |
| No dead code, unused imports, hanging TODOs | `ruff` rules `F401`, `F841`, `T201`, `FIX002` (TODO without issue link) enabled |
| Static analysis findings triaged, not ignored | `mypy` and `ruff` configured to fail CI on any unsuppressed finding; suppressions require inline `# noqa: <rule> — reason` |
| Peer review with constitution-compliance sign-off | PR template (added in Phase 2 task setup) carries the four-gate checklist |

**Verdict**: PASS.

### Gate 2 — Test-Driven Development (NON-NEGOTIABLE)

| Constitution requirement | Plan compliance |
|--------------------------|-----------------|
| Tests written before production code (visible in commits) | Phase-2 tasks (created by `/speckit-tasks`) will be ordered tests-first per user story; commit policy will require a failing test commit before its implementation commit |
| Every FR-* and acceptance scenario maps to ≥ 1 test | Contract tests in `tests/contract/` cover the public API surface; spec→test mapping recorded in `tests/contract/spec_mapping.md` (created in Phase 2) |
| Coverage ≥ 80% line / ≥ 70% branch on new code | `pytest --cov=src --cov-fail-under=80 --cov-branch` plus a branch-coverage assertion in CI |
| Tests deterministic; flaky tests quarantined within 24h | All randomness routed through `hypothesis` (seeded) or fixed RNGs; CI re-runs disabled — failure is failure |
| Integration tests for every public API contract | `tests/contract/` is mandatory for the API surface defined in `contracts/api.md`; `tests/integration/` covers cross-module flows with real GeoTIFF fixtures |

**Verdict**: PASS.

### Gate 3 — Consistency

| Constitution requirement | Plan compliance |
|--------------------------|-----------------|
| One coding style, enforced by tooling | `pyproject.toml` carries the single source of truth for `ruff`, `mypy`, `pytest` config |
| Naming/layout follow the patterns recorded here | Project Structure (below) is the canonical layout; deviations require Complexity Tracking entries |
| Uniform error format, logging, config | One exception hierarchy under `errors.ElevationError`; all logs via `logging.getLogger("us_elevation_profile.<module>")` with structured key=value formatter |
| New dependencies justified in writing | Each of the 7 runtime/dev dependencies above has its rationale inline |
| Consistent terminology across docs, errors, logs | The Key Entities glossary in the spec (Elevation Store, Tile, Endpoint, Profile Sample, Path Profile) is the canonical vocabulary; same names used in code, docstrings, error messages, and log records |

**Verdict**: PASS.

### Gate 4 — Performance Requirements

| Constitution requirement | Plan compliance |
|--------------------------|-----------------|
| Performance budgets declared in plan.md | The Performance Goals table above carries five concrete budgets, one per SC-* perf criterion |
| Hot-path changes carry automated benchmarks | `tests/benchmarks/` contains pytest-benchmark suites for every budget; benchmarks run on every PR via CI |
| > 5% regression blocks merge | `pytest-benchmark compare --fail-on-regression-percent 5` against the main-branch baseline |
| Big-O documented for non-trivial routines | Required for: bilinear interpolation, geodesic sampling, LRU cache eviction, bbox lookup; documented inline in each module |
| Observability sufficient to validate budgets | `logging` module-level loggers + per-operation timing counters; benchmark harness asserts timings against budgets |

**Verdict**: PASS.

**Overall Constitution Check**: PASS. No violations; Complexity Tracking section
remains empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-us-elevation-profile/
├── plan.md                # This file
├── research.md            # Phase 0: technical decisions consolidated
├── data-model.md          # Phase 1: entity → field mapping with validation rules
├── quickstart.md          # Phase 1: ≤ 10-min onboarding (SC-005)
├── contracts/
│   └── api.md             # Phase 1: public Python API surface contract
├── checklists/
│   └── requirements.md    # From /speckit-specify + /speckit-clarify
└── tasks.md               # Created later by /speckit-tasks
```

### Source Code (repository root)

```text
src/
└── us_elevation_profile/
    ├── __init__.py        # Public API re-exports
    ├── api.py             # High-level functions: register_dataset(), elevation_at(), profile_between()
    ├── store.py           # ElevationStore class + SQLite sidecar index (FR-001, FR-002, FR-017, FR-018)
    ├── tile.py            # Tile abstraction; rasterio windowed reads (FR-014, FR-010)
    ├── cache.py           # LRU working-set cache of decoded tile arrays (FR-015)
    ├── interpolate.py     # Bilinear interpolation with no-data propagation (FR-019)
    ├── path.py            # Geodesic line sampling on WGS84 (FR-005, FR-006); range validation (FR-016)
    ├── profile.py         # PathProfile, ProfileSample, Endpoint dataclasses (FR-009, FR-012)
    └── errors.py          # ElevationError hierarchy: OutOfCoverageError, OutOfRangeError, etc.

tests/
├── contract/              # Public API contract tests (one per public symbol)
│   ├── test_register_dataset.py
│   ├── test_elevation_at.py
│   ├── test_profile_between.py
│   └── spec_mapping.md    # FR-*/SC-* → test-id mapping (Principle II)
├── integration/           # End-to-end flows with real (sample) GeoTIFF fixtures
│   ├── test_register_and_query.py
│   ├── test_cross_tile_profile.py
│   └── test_persistent_index.py
├── unit/                  # Pure-unit tests, no IO
│   ├── test_bilinear.py
│   ├── test_geodesic_sampling.py
│   ├── test_lru_cache.py
│   └── test_input_validation.py
├── benchmarks/            # pytest-benchmark suites mapped to SC-001/002/008/009
│   ├── bench_point_query.py
│   ├── bench_profile_short.py
│   ├── bench_profile_long.py
│   ├── bench_registration.py
│   └── bench_restart.py
└── data/                  # Small synthetic GeoTIFF fixtures (< 10 MB total)
    └── synthetic_1deg_*.tif

pyproject.toml             # Single source of truth for build, deps, tool configs
README.md                  # Project landing
CLAUDE.md                  # Agent guidance (existing)
.specify/                  # Speckit workflow assets (existing)
```

**Structure Decision**: Standard Python `src/`-layout single-package library. Keeps
package isolated from the repo root (avoids accidental imports from tests during
development); aligns with PEP 621 and modern Python packaging conventions; matches
Principle III consistency (one canonical layout). All modules are flat under the
package — no nested sub-packages in v1, keeping the import surface and review surface
small.

## Complexity Tracking

> Fill ONLY if Constitution Check has violations that must be justified.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| — none — | — | — |

All four constitution gates pass on the design above. No principles are violated, so
no entries are required.
