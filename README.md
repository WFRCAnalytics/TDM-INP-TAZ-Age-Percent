# TDM-EST-TAZ-Age-Percent Methodology

**Goal:** Estimate age‑group shares (0–17, 18–64, 65+) for each USTM v4 TAZ using 2020 Census blocks, 2023 ACS age structure, and spatial splits, with sensible fallbacks and QC.

1. **Inputs & setup**

   * Geometry: 2020 Census blocks for Utah; USTM v4 TAZ polygons (must include `CO_TAZID`); USTM v3 Medium Districts (`DISTMED`) for temporary district mapping.
   * Census data:

     * 2020 Decennial PL at **block**: `P1_001N` (total pop), `P3_001N` (18+ pop).
     * 2023 ACS 5‑year **B01001** at **block group** and **tract** to derive 0–17, 18–64, 65+ counts and `%65+ of 18+`.
   * CRS: Reproject as needed; use a projected UTM CRS for accurate areas/centroids.

2. **Assign Medium Districts to TAZs**

   * Compute TAZ centroids in a projected CRS.
   * Spatially join centroids to v3 Medium District polygons to attach `DISTMED` to each TAZ.
   * Join `DISTMED` back to the original TAZ polygons.

3. **Create spatial splits**

   * **Block→TAZ splits:** Overlay blocks with TAZs; compute intersection areas.
   * Filter out slivers: drop intersected pieces < **1%** of the **original block area**.
   * For each block, recompute the **remaining total area** and normalize each piece to get `blk_area_ratio` that sums to 1 across its kept pieces.
   * Parse GEOIDs into `state`, `county`, `tract`, `blk`, `blkgrp`.
   * **Tract→TAZ splits:** Aggregate intersected areas to tract–TAZ, compute `tract_area_ratio` by normalizing within each tract.

4. **Fetch and prepare demographic inputs**

   * 2020 PL at **block** for all Utah counties: total population and 18+ population.
   * 2023 ACS B01001:

     * **Block group level:** compute `Pop_0to17`, `Pop_18to64`, `Pop_65plus`; derive `acs_blkgrp_pct65PlusOf18Plus = Pop_65plus / (Pop_65plus + Pop_18to64)`.
     * **Tract level:** same aggregation to get `acs_tract_pct65PlusOf18Plus`.

5. **Estimate age detail on split geographies**

   * **Block‑split estimates (preferred source):**

     * Allocate **total** and **18+** block population to each block–TAZ piece using `blk_area_ratio`.
     * Apply the **block‑group** `%65+ of 18+` to the allocated 18+ to estimate 65+; derive 18–64 as residual of 18+; derive 0–17 as `total − 18+`.
     * Enforce a single county label per TAZ by replacing piece‑level `county` with the **mode** county within each `CO_TAZID`.
   * **Tract‑split estimates (fallback):**

     * Allocate **tract‑level** total and 18+ populations to tract–TAZ using `tract_area_ratio`.
     * Apply the **tract‑level** `%65+ of 18+` to estimate 65+; derive 18–64 and 0–17 as above.
     * Apply the same county‑mode enforcement per `CO_TAZID`.

6. **Aggregate to reporting geographies & set source precedence**

   * **By TAZ (block source):** Sum block‑split estimates to `CO_TAZID` × county; keep TAZs with total estimated persons ≥ **50**; tag `SOURCE = "block"`.
   * **By TAZ (tract fallback):** For any `CO_TAZID` **not present** in the block result, sum tract‑split estimates; tag `SOURCE = "tract"`.
   * **By Medium District fallback:** Additionally aggregate block‑split estimates by `DISTMED` × county, then map back to TAZs via (`DISTMED`,`CO_FIPS`) for any remaining `CO_TAZID` missing from both block and tract; tag `SOURCE = "distmed"`.
   * Concatenate all sources and sort by `CO_TAZID`.

7. **Convert to percentages & quality checks**

   * Compute `PCT_0TO17`, `PCT_18TO64`, `PCT_65P` as shares of (0–17 + 18–64 + 65+).
   * Round to two decimals with a final small adjustment so the **row sum equals exactly 1.00** (`PCT_SUM` check).
   * Verify no duplicate `CO_TAZID` remain after source precedence.
   * Keep `CO_FIPS` (county FIPS), `SOURCE`, and `;CO_TAZID` (Excel‑friendly leading semicolon).

8. **Output**

   * Write `results/Lookup - BYTAZAgePct - AllCo.csv` with columns:

     * `;CO_TAZID`, `PCT_SUM`, `PCT_0TO17`, `PCT_18TO64`, `PCT_65P`, `SOURCE`, `CO_FIPS`.

**Notes & assumptions**

* Sliver threshold = **1%** of the original block area (applied before normalizing proportions).
* Population thresholds: exclude TAZ rows with fewer than **50** estimated persons (per source path) to reduce instability.
* ACS year = **2023** to reflect recent age structure; 2020 used for cross‑checks where noted.
* Medium District mapping is temporary (v3 districts) until districts are assigned directly to the new TAZs.
