# Quickstart: `us_elevation_profile`

**Audience**: a developer who has never used this library before.
**Goal**: load a small elevation dataset and compute your first transmitter→receiver
profile in **under 10 minutes** (SC-005).

This guide assumes Python 3.11+ on Linux x86_64. macOS works the same way; Windows
is untested in v1.

---

## 1. Install (1 min)

```bash
pip install us-elevation-profile
# or, from a clone:
pip install -e .[dev]
```

The wheel pulls in `rasterio`, `numpy`, and `pyproj` from PyPI.

---

## 2. Get some elevation tiles (3 min)

This library reads **GeoTIFF NED tiles** as distributed by USGS 3DEP. For a quick
start, you can download a single 1° × 1° tile covering an area you care about from
[https://apps.nationalmap.gov/downloader/](https://apps.nationalmap.gov/downloader/)
(select **Elevation Products → 1 arc-second DEM → GeoTIFF**). Drop the `.tif` files
into a directory, for example `~/elevation/`.

For a self-contained test you can instead point the library at the synthetic fixtures
that ship with the test suite — see `tests/data/` in the repository.

---

## 3. Register the dataset (1 min)

```python
from us_elevation_profile import ElevationStore

store = ElevationStore("~/elevation/")
summary = store.register_directory("~/elevation/")
print(summary)
# RegistrationSummary(added=N, updated=0, removed=0, skipped=0, ...)
```

`register_directory` scans for `*.tif`, opens each one to read its bbox and
resolution, and writes a small sidecar file (`.us_elevation_profile_index.sqlite`)
next to your data so subsequent runs reload in seconds (SC-009).

For the full target dataset (~1,756 files) this takes **under 5 minutes** the first
time (SC-008) and **under 5 seconds** thereafter.

---

## 4. Query a single elevation (30 sec)

```python
# Mount Whitney summit, California
elev_m = store.elevation_at(latitude=36.5785, longitude=-118.2923)
print(f"{elev_m:.1f} m")
# 4421.2 m
```

If the point is outside any registered tile, the call raises `OutOfCoverageError`.
If the point lands on a no-data cell (oceans, voids) the return is `NaN`.

---

## 5. Compute a transmitter→receiver profile (2 min)

```python
from us_elevation_profile import Endpoint

tx = Endpoint(latitude=37.7749, longitude=-122.4194, label="transmitter")   # San Francisco
rx = Endpoint(latitude=37.8044, longitude=-122.2712, label="receiver")      # Oakland

profile = store.profile_between(tx, rx)
print(f"length = {profile.total_length_m / 1000:.2f} km, "
      f"samples = {len(profile.samples)}, "
      f"no_data_segments = {len(profile.no_data_segments)}")
# length = 13.04 km, samples = 1304, no_data_segments = 0
```

To plot the profile (requires matplotlib, not a dependency of this library):

```python
import matplotlib.pyplot as plt

x_km = profile.samples["distance_m"] / 1000.0
y_m  = profile.samples["elevation_m"]      # may contain NaN
plt.plot(x_km, y_m)
plt.xlabel("Distance from transmitter (km)")
plt.ylabel("Elevation (m)")
plt.show()
```

NaN samples render as gaps in the plot, which is the intended behaviour.

---

## 6. Common knobs (1 min)

- **Custom sample spacing**:

  ```python
  store.profile_between(tx, rx, spacing_m=10.0)   # one sample every 10 m
  ```

- **Fixed sample count** (mutually exclusive with `spacing_m`):

  ```python
  store.profile_between(tx, rx, num_samples=1024)
  ```

- **Memory budget** at store construction:

  ```python
  store = ElevationStore("~/elevation/", working_set_bytes=512 * 1024 * 1024)  # 512 MiB
  ```

- **Context manager** for clean shutdown:

  ```python
  with ElevationStore("~/elevation/") as store:
      profile = store.profile_between(tx, rx)
  ```

---

## 7. What can go wrong (1 min)

| You see | What it means | Fix |
|---------|--------------|-----|
| `InvalidDatasetRootError` | Dataset directory does not exist. | Check the path. |
| `InvalidCoordinateError` | Latitude not in [-90, 90] or longitude not in [-180, 180]. | Fix the input. |
| `OutOfRangeError(side="below")` | tx–rx distance < 1 km. | Use a longer path or v2 of the library. |
| `OutOfRangeError(side="above")` | tx–rx distance > 200 km. | Use a shorter path or v2 of the library. |
| `OutOfCoverageError` (from `elevation_at`) | No registered tile covers the point. | Add the relevant tile(s) and call `register_directory` again. |
| Profile shows `no_data_segments` | Path crosses ocean / voids / unregistered area. | Expected; inspect the segments to decide whether to extend coverage. |
| `StaleIndexError` | Index references files that disappeared and refresh failed. | Re-run `register_directory` to rebuild. |

All errors inherit from `ElevationError`, so `except ElevationError` is a safe
catch-all if you need one.

---

## 8. Next steps

- The full spec: [`spec.md`](./spec.md).
- The frozen public API contract: [`contracts/api.md`](./contracts/api.md).
- For RF-specific computation (line-of-sight, Fresnel zone, link budgets), pull the
  raw `profile.samples` array out and run your own analysis — this library
  intentionally stops at the elevation profile (see spec Assumptions).
