# Feature Specification: US Elevation Profile Module

**Feature Branch**: `001-us-elevation-profile`

**Created**: 2026-05-13

**Status**: Draft

**Input**: User description: "I want a module in python to store US elevation data files and between two points of latitude, longitude - of receiver and transmitter"

## Clarifications

### Session 2026-05-13

- Q: Dataset scale the module must handle → A: Full CONUS at native high resolution, **plus US territories** (notably Hawaii and Puerto Rico). Target dataset for v1 is approximately **1,756 NED tile files** comprising **~23 billion elevation samples**, each stored as a **64-bit float** (≈ 180 GB raw if fully resident).
- Q: Concrete NED file format(s) supported in v1 → A: **GeoTIFF only** (`.tif`). The module loads USGS 3DEP-style single-file GeoTIFF tiles; other historical NED encodings (GridFloat, ArcGrid, IMG, etc.) are out of scope for v1.
- Q: Path-length range the module must serve → A: **1 km to 200 km** great-circle distance between transmitter and receiver. Endpoint pairs outside this range are out of supported scope for v1 and are rejected with a clear out-of-range error.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Compute elevation profile between transmitter and receiver (Priority: P1)

A wireless link planner has the geographic coordinates of a fixed transmitter and a candidate receiver site in the United States. They need to inspect the terrain between the two points to judge whether the path is viable (for example, to reason about line-of-sight or terrain occlusion). The module accepts the two coordinate pairs and returns a sampled elevation profile along the path connecting them.

**Why this priority**: This is the core capability the user is asking for. Without it, the module has no externally visible value; the storage layer (US2) exists to support this query.

**Independent Test**: With a small pre-loaded elevation dataset covering a known region, request a profile between two known points and verify that the returned profile is non-empty, ordered by distance along the path, and contains elevations consistent with the source data at sampled points.

**Acceptance Scenarios**:

1. **Given** elevation data covering both endpoints has been loaded, **When** the user requests a profile between a transmitter at (lat₁, lon₁) and a receiver at (lat₂, lon₂), **Then** the module returns an ordered series of sample points, each with the distance along the path and the elevation at that location.
2. **Given** a request with default sampling, **When** the path length is L kilometres, **Then** the number of samples returned is appropriate for the underlying data resolution and L, sufficient to reflect terrain features visible in the source data.
3. **Given** a request specifies a custom sample count or sample spacing, **When** the parameters are valid, **Then** the returned profile uses the requested sampling.

---

### User Story 2 - Load and organise US elevation data (Priority: P1)

The user has obtained a collection of US elevation data files (for example, tiles covering a region of interest) and needs the module to manage them: register them with a store, index them by geographic coverage, and serve them on demand to the profile and point-query operations.

**Why this priority**: Profile computation (US1) cannot return useful results without elevation data. Loading and organisation are the foundation US1 depends on; both ship together as the MVP.

**Independent Test**: Provide a directory containing a small set of elevation files. Verify the module reports them as loaded, exposes their combined geographic coverage, and successfully serves elevation queries for points inside that coverage.

**Acceptance Scenarios**:

1. **Given** a directory of supported elevation files, **When** the user registers it with the module, **Then** all files are catalogued and the module reports the combined coverage area.
2. **Given** a file is malformed or in an unsupported format, **When** loading is attempted, **Then** that file is skipped with a clear diagnostic and the remaining files load successfully.
3. **Given** elevation data has been loaded, **When** the user requests the elevation at a specific (lat, lon) within coverage, **Then** the module returns the elevation value sourced from the appropriate tile.

---

### User Story 3 - Handle queries outside the loaded coverage (Priority: P2)

A user requests a profile or a point elevation where one or both points fall outside the data the module has been given. The module must respond with a clear, actionable signal rather than failing silently or returning misleading values.

**Why this priority**: Predictable behaviour at the boundary of the data is required for the module to be safe to integrate, but the happy path is usable before this is hardened.

**Independent Test**: Issue a query for a coordinate outside the loaded coverage and verify that the response cleanly indicates "no data" rather than crashing, returning zero, or silently substituting a nearby value.

**Acceptance Scenarios**:

1. **Given** loaded coverage that excludes the requested point, **When** the user requests a single-point elevation, **Then** the response signals that the point is outside coverage.
2. **Given** a path where one endpoint is outside coverage, **When** the user requests a profile, **Then** the response signals the partial-coverage condition and identifies which segments of the path lack data.

---

### Edge Cases

- Both endpoints fall inside the same source tile (very short path).
- Path crosses a tile boundary (samples must remain consistent across the seam).
- One or both endpoints land in a no-data cell (for example, water bodies that DEMs mask out).
- Endpoint pair distance is just below 1 km or just above 200 km — the module MUST reject with an out-of-range error that identifies which boundary was violated (per FR-016).
- Endpoints are identical (path length = 0) — falls below the 1 km minimum and is rejected.
- Coordinates are valid floats but lie entirely outside the United States.
- Latitude or longitude is outside its valid global range (for example, latitude > 90).
- A previously loaded source file is moved or deleted between registration and query.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The module MUST allow a user to register one or more local sources of US elevation data files and make those files available for elevation queries. The supported source format for v1 is single-file GeoTIFF (`.tif`) NED tiles as distributed by USGS 3DEP; other NED encodings are out of scope for v1.
- **FR-002**: The module MUST report the geographic coverage of the currently loaded data (for example, a bounding region or the list of covered tiles).
- **FR-003**: The module MUST return the elevation at a given (latitude, longitude) when that point is within loaded coverage.
- **FR-004**: The module MUST return an ordered, sampled elevation profile between two given (latitude, longitude) endpoints, in which each sample carries its distance along the path and its elevation.
- **FR-005**: The path between two endpoints MUST follow the geodesic (great-circle) line on the WGS84 ellipsoid.
- **FR-006**: The module MUST accept an optional sampling parameter (sample count or sample spacing) for profile requests and apply a sensible default when none is supplied.
- **FR-007**: The module MUST validate inputs and reject coordinates whose latitude is outside [-90, 90] or whose longitude is outside [-180, 180] with a clear error.
- **FR-008**: The module MUST clearly distinguish "point outside loaded coverage" from a valid elevation result in its responses.
- **FR-009**: When a profile spans regions of partial coverage, the module MUST indicate which segments lack data rather than silently substituting values.
- **FR-010**: The module MUST tolerate malformed or unreadable data files by skipping them with a diagnostic, without aborting the load of the remaining files.
- **FR-011**: Elevation values returned MUST be expressed in metres above mean sea level, consistent with the underlying source data.
- **FR-012**: The module MUST label the two endpoints of a profile request as transmitter and receiver, and preserve that labelling in the response so callers can orient the profile (distance is measured from transmitter to receiver).
- **FR-013**: The module MUST support a fully registered dataset on the order of ~1,800 NED source files and ~23 billion 64-bit elevation samples (≈ 180 GB raw) without requiring the full dataset to be resident in memory at any time.
- **FR-014**: The module MUST resolve any single-point or profile query by reading only the tile(s) intersected by the query; loading all registered tiles eagerly for a query is forbidden.
- **FR-015**: The module MUST bound resident memory attributable to elevation data by a configurable working-set budget. Beyond that budget, less-recently-used tiles MUST be evicted; resident memory MUST NOT grow with total registered dataset size.
- **FR-016**: The module MUST validate that the great-circle distance between the transmitter and receiver endpoints is between 1 km and 200 km inclusive, and reject out-of-range profile requests with a clear, typed error indicating which side of the range was violated.

### Key Entities

- **Elevation Store**: The collection of registered US elevation data (CONUS + supported territories), with a known coverage extent, a tile index that survives across sessions, and the ability to serve point queries against a dataset that may contain on the order of ~1,800 source tiles and tens of billions of samples without loading the full dataset into memory.
- **Tile**: A single source file of elevation data covering a rectangular geographic region with a defined grid resolution.
- **Endpoint**: A geographic point identified by latitude and longitude, labelled in profile requests as either the transmitter or the receiver.
- **Profile Sample**: One observation along the path between two endpoints, carrying its position on the path (distance from the transmitter, latitude, longitude) and the elevation at that position or a no-data indicator.
- **Path Profile**: The ordered series of profile samples produced for one transmitter–receiver pair, with overall metadata (total length, sample count, any partial-coverage indication).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For a path of length between 1 km and 200 km between two endpoints fully within loaded coverage, the module returns the elevation profile in under 2 seconds on a typical developer workstation.
- **SC-002**: For a single-point query within loaded coverage, the module returns the elevation in under 50 milliseconds on a typical developer workstation.
- **SC-003**: Elevation values returned for sampled points agree with the source dataset to within the published vertical accuracy of that dataset.
- **SC-004**: 100% of valid input requests result in either an elevation or profile response, or a clearly typed "outside coverage" or "no data" signal; no valid input causes an unhandled error.
- **SC-005**: A developer following the quickstart can load a sample dataset and compute their first profile in under 10 minutes.
- **SC-006**: When a profile crosses a tile boundary, sampled elevations on either side of the seam differ by no more than the natural terrain variation between the two adjacent samples; the storage layer introduces no visible discontinuity.
- **SC-007**: With the full target dataset registered (~1,756 NED files, ~23 billion samples), the module's resident memory attributable to elevation tiles stays within the configured working-set budget and does not grow with the size of the registered dataset.
- **SC-008**: Registering the full target dataset (~1,756 NED files) completes in under 5 minutes on a typical developer workstation and produces an index that supports subsequent point/profile queries without re-scanning all source files.

## Assumptions

- Inputs are expressed in WGS84 decimal-degree latitude and longitude; elevations are returned in metres above mean sea level.
- Coverage area is the Continental United States plus US territories supported by the USGS National Elevation Dataset (NED), notably Hawaii and Puerto Rico. The target dataset for v1 is approximately 1,756 NED tile files / ~23 billion 64-bit elevation samples (see Clarifications).
- The module manages elevation data that the user has already obtained externally (for example, from public sources). Automatic discovery, download, or refresh of source datasets is out of scope for v1.
- The "transmitter" and "receiver" labels are conventions for the two endpoints; the v1 scope is the elevation profile itself. RF-specific computations (line-of-sight assessment, Fresnel-zone clearance, link budgets) are out of scope and can be derived by callers from the profile or added in a later iteration.
- The module is delivered as an importable library used from caller code; no separate service or command-line tool is in scope for v1.
- Source elevation files are single-file GeoTIFF NED tiles as distributed by USGS 3DEP (confirmed in Clarifications). Other NED encodings (GridFloat, ArcGrid, IMG, etc.) and non-NED elevation products are out of scope for v1.
- Disk usage scales with the size of the registered dataset (full v1 target ≈ 180 GB of raw 64-bit values; less when source files are compressed). Resident memory is bounded by a working-set budget, not by dataset size — see FR-013 through FR-015.
