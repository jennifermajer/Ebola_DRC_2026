# Health-zone road travel time and distance matrices

Pairwise **driving** travel times and road distances between all health zones in the Democratic Republic of the Congo (DRC), derived from zone polygons in `data/shapefiles/DRC_Health_zones.shp`.

These matrices support connectivity analyses for outbreak modelling (e.g. gravity-style coupling, accessibility indices, or validation of reported movement patterns).

---

## Files

| File | Description |
|------|-------------|
| `RDC_Full_Travel_Time_Minutes.csv` | Origin–destination matrix of **car** travel time (minutes) |
| `RDC_Full_Road_Distance_Km.csv` | Origin–destination matrix of **car** road distance (kilometres) |
| `calculate_healthzone_travel_time_and_distance.R` | Script that builds the matrices via the [OSRM](http://project-osrm.org/) routing engine |

**Dimensions:** 519 × 519 health zones (~2.7M origin–destination pairs per matrix).  
**Size:** ~1.7 MB (time), ~2.2 MB (distance).

---

## Method

1. **Zones** — Load `DRC_Health_zones.shp` (519 polygons). Zone labels use the shapefile field `Nom` (health-zone name), matching naming elsewhere in this repository.
2. **Representative points** — Reproject to WGS84 (EPSG:4326) and take `st_point_on_surface()` for each polygon so the routing origin/destination lies inside the zone (centroids can fall outside irregular polygons).
3. **Routing** — Query the public OSRM Table API (`osrm` R package, `osrm.profile = "car"`) for all pairs of representative points.
4. **Batching** — Requests are split into 20×20 zone chunks (400 pairs per call) with a 1-second pause between batches to reduce load on the public OSRM server. Failed batches leave `NA` for the affected cells.
5. **Export** — Square matrices are written as CSV with zone names as row labels (first column) and column headers.

**Units**

- Travel time: **minutes** (as returned by the `osrm` package’s `osrmTable()` duration output).
- Distance: OSRM returns metres; the script divides by 1000 before saving (**kilometres**).

**What this is not**

- Times and distances are **modelled road network** estimates, not observed travel data.
- Routing uses the **car** profile only (no walking, ferry-specific, or seasonal road logic).
- Results depend on the OSM road network underlying the public OSRM instance at the time the script was run.

---

## CSV format

Both files share the same layout:

- **First column** (`""` in the header): origin zone name (`Nom`).
- **Remaining columns:** destination zone names, in the same order as rows.
- **Cell `(i, j)`:** travel time or distance from zone `i` to zone `j`.
- **Diagonal:** `0` (same zone).
- **Symmetry:** Times and distances are not guaranteed to be symmetric (one-way systems, turn restrictions, and OSRM’s directed graph can yield small differences); treat matrices as directed unless you explicitly symmetrise.

**Example (reading in R):**

```r
library(here)
dur <- read.csv(here("data/healthzone_traveldisttime/RDC_Full_Travel_Time_Minutes.csv"),
                row.names = 1, check.names = FALSE)
dist_km <- read.csv(here("data/healthzone_traveldisttime/RDC_Full_Road_Distance_Km.csv"),
                    row.names = 1, check.names = FALSE)

# Travel time from "Goma" to "Beni" (minutes)
dur["Goma", "Beni"]

# Road distance from "Goma" to "Beni" (km)
dist_km["Goma", "Beni"]
```

Use `check.names = FALSE` when loading so column names are not altered. See **Limitations** if you use `row.names = 1`.

---

## Regenerating the data

From the **repository root** (the script uses `here()` and expects the project layout below):

```bash
Rscript data/healthzone_traveldisttime/calculate_healthzone_travel_time_and_distance.R
```

**Requirements:** R packages `sf`, `dplyr`, `osrm`, `tictoc`, `here`; network access to the OSRM API.

**Runtime:** On the order of **hours** for the full country (roughly `(519 / 20)² ≈ 676` batched table calls, plus 1 s sleep per batch). Progress is printed to the console; total elapsed time is reported by `tictoc` at the end.

Outputs are overwritten in this directory:

- `RDC_Full_Travel_Time_Minutes.csv`
- `RDC_Full_Road_Distance_Km.csv`

---

## Data quality and limitations

| Issue | Detail |
|-------|--------|
| **Duplicate zone names** | Two names each appear twice in the shapefile: **Bili** and **Lubunga** (four rows, two unique labels). The CSV contains separate rows with identical names but **different** values. Do not rely on name-only indexing; use a unique zone ID from the shapefile if you need unambiguous joins. |
| **`row.names` in R** | `read.csv(..., row.names = 1)` will fail or drop rows because of duplicate names. Prefer reading the first column explicitly and keeping a shapefile-based ID, or use `check.names = FALSE` without `row.names`. |
| **Idjwi** | The zone **Idjwi** (island in Lake Kivu) has **518 missing** values in the time matrix—likely because the public OSRM **car** network does not connect the island to the mainland in the extracted graph. Treat Idjwi-related flows with caution or impute separately. |
| **Other `NA` cells** | Aside from Idjwi, most zones have a single missing value (often corresponding to failed pairs involving Idjwi or API errors). Failed batches are logged in the console and left as `NA`. |
| **Coverage** | Values span roughly **0–4,342 minutes** (~72 h) and **0–4,118 km** road distance for the longest cross-country pairs in the current extract. |
| **Public API** | Re-running the script may yield slightly different results if OSM/OSRM data are updated. Heavy use may trigger rate limiting (`429`); the script sleeps 1 s between batches to mitigate this. |

---

## Provenance

- **Geometry:** `data/shapefiles/DRC_Health_zones.shp` (see `data/shapefiles/README.md`).
- **Routing engine:** [OSRM](http://project-osrm.org/) via the [`osrm`](https://cran.r-project.org/package=osrm) R package (`osrmTable`, `car` profile).
- **Script date in repo:** `calculate_healthzone_travel_time_and_distance.R` (May 2026).
- **CSV timestamp:** March 2026 (as on disk); regenerate if you need consistency with a newer road network.

For questions about how these matrices are used in the wider project, see the root `README.md` and `data/README.md`.
