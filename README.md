# NDVI-Based Early Vegetation & Salinity Zoning — **Elalab** & **Cafine**

This repository implements a reproducible workflow to detect the **first appearance of vegetation** from multi-temporal **NDVI** and use it as a proxy for **soil salinity**. Pixels that green up earlier tend to sit over **low-salinity (“sweet”) soils**, while delayed greening suggests higher salinity. The **same layout and methods** are applied to two study areas: **Elalab** and **Cafine**.

---

## Sensors & Bands
- **Elalab (2021, 2022, 2024)** — Planet (≥4 bands): **Red = band 3 (index 2)**, **NIR = band 4 (index 3)**.  
- **Cafine (2019, 2022, 2023)** — Sentinel‑2 for **2019** (2 bands): **B4 = Red (index 0)**, **B8 = NIR (index 1)**; Planet for **2022–2023** as above.  
- All rasters are named `YYYY_MM_DD.tif` and processed chronologically.

---

## Folder Structure (shared)
```
Change_detection_AM/
├─ Codes/
│  ├─ elalab_flow.py
│  └─ cafine_flow_simple.py
├─ Data/
│  ├─ Images/
│  │  ├─ Elalab/
│  │  │  ├─ 2021/*.tif
│  │  │  ├─ 2022/*.tif
│  │  │  └─ 2024/*.tif
│  │  └─ Cafine_Cafal/
│  │     ├─ 2019_sentinel/*.tif
│  │     ├─ 2022/*.tif
│  │     └─ 2023/*.tif
│  └─ Shapefiles/
│     ├─ Elalab_salt_UTM.shp    # CE (electrical conductivity)
│     ├─ Cafine_salt_UTM.shp    # CE (electrical conductivity)
│     ├─ Elalab_AOI.shp         # Area of Interest (Elalab)
│     └─ Cafine_AOI.shp         # Area of Interest (Cafine)
├─ Images/
│  ├─ Elalab/
│  │  ├─ violins_2021/*.png
│  │  ├─ violins_2022/*.png
│  │  ├─ violins_2024/*.png
│  │  └─ violins_final/violins_final_thr_XX.png
│  └─ Cafine_Cafal/
│     ├─ violins_2019/*.png
│     ├─ violins_2022/*.png
│     ├─ violins_2023/*.png
│     └─ violins_final/violins_final_thr_XX.png
└─ Results/
   ├─ Elalab/
   │  ├─ 2021/Elalab_2021_earlyveg_thr_XX.tif
   │  ├─ 2022/Elalab_2022_earlyveg_thr_XX.tif
   │  ├─ 2024/Elalab_2024_earlyveg_thr_XX.tif
   │  ├─ 2021/Elalab_2021_earlyveg_FINAL_thr_XX.tif
   │  ├─ 2022/Elalab_2022_earlyveg_FINAL_thr_XX.tif
   │  ├─ 2024/Elalab_2024_earlyveg_FINAL_thr_XX.tif
   │  ├─ associated_mangrove_consensus_thr_XX.tif
   │  ├─ associated_mangrove_consensus_thr_XX.shp
   │  ├─ associated_mangrove_consensus_thr_XX_clean.tif   # after island removal
   │  ├─ associated_mangrove_consensus_thr_XX_clean.shp   # after island removal
   │  ├─ sensitivity_vs_tau.png                           # Cliff’s δ (0–100 scaled)
   │  ├─ area_vs_tau.png                                  # area fraction vs τ
   │  └─ AM_points_summary_thr_XX.xlsx
   └─ Cafine_Cafal/
      ├─ 2019/Cafine_Cafal_2019_earlyveg_thr_XX.tif
      ├─ 2022/Cafine_Cafal_2022_earlyveg_thr_XX.tif
      ├─ 2023/Cafine_Cafal_2023_earlyveg_thr_XX.tif
      ├─ 2019/Cafine_Cafal_2019_earlyveg_FINAL_thr_XX.tif
      ├─ 2022/Cafine_Cafal_2022_earlyveg_FINAL_thr_XX.tif
      ├─ 2023/Cafine_Cafal_2023_earlyveg_FINAL_thr_XX.tif
      ├─ associated_mangrove_consensus_thr_XX.tif
      ├─ associated_mangrove_consensus_thr_XX.shp
      ├─ associated_mangrove_consensus_thr_XX_clean.tif
      ├─ associated_mangrove_consensus_thr_XX_clean.shp
      ├─ sensitivity_vs_tau.png
      ├─ area_vs_tau.png
      └─ AM_points_summary_thr_XX.xlsx
```

---

## Workflow (per area)
1. **Per‑year threshold sweep.** For each year, evaluate NDVI thresholds:  
   **τ ∈ [0.15, 0.50] (step 0.01)** with rule **ΔNDVI > 0.10 & NDVIₜ > τ**.  
   At each τ:
   - Detect **early appearance** and save the **binary raster** (1 = at least one early event in the year).
   - Spatially join CE sample points to the mask and produce **violin plots** (CE IN vs CE OUT).
   - Write a **metrics CSV** (`*_threshold_sensitivity.csv`) with: `tau`, `n_in`, `n_out`, **`area_frac` inside AOI**, **`med_diff`** (median(OUT)−median(IN)), **`cliffs_delta`** (effect size).

2. **Manual threshold selection (per area).** After reviewing plots/metrics, set `CHOSEN_THR` (e.g., **0.26**). Use a distinct τ per area if needed.  
   - The **sensitivity figure** now shows **Cliff’s delta (δ)** **per year**, **normalized to a 0–100 scale** (robust 5–95% range). The **red line** is the **inter‑annual mean** and the band is **±1σ**. This summarizes how consistently **ECe(OUT) exceeds ECe(IN)** across thresholds and years.

3. **Apply the chosen threshold per year.** With `CHOSEN_THR`, recompute and save **final rasters** for each year.

4. **Final overlap (AM).** **Align** final rasters to a reference grid and take the **intersection (3/3)** to obtain the **Associated Mangrove (AM)**.  
   **Island removal is mandatory:** polygons **< 1000 m²** are removed; both **raw** and **_clean** consensus files are written.

5. **Validation.** Compare CE distributions **IN_AM** vs **OUT_AM** (medians, IQR, *n*, Cliff’s δ) and export a **final violin**. Visual checks against imagery and field context are recommended.

6. **Tables & map.** Export **Excel** (`AM_points_summary_thr_XX.xlsx`) with sheets **ALL**, **IN_AM**, **OUT_AM**; notebooks can display an inline **Folium** map with AM and colored points.

---

## NDVI, Early Appearance & Threshold Selection

### NDVI per date
At date *t*:  
**NDVIₜ = (NIRₜ − Redₜ) / (NIRₜ + Redₜ)**

### First appearance (per‑pixel rule)
Let **Δₜ(x) = NDVIₜ(x) − NDVIₜ₋₁(x)**. Pixel *x* exhibits first appearance at the earliest *t* such that:  
**ΔNDVI > 0.10** and **NDVIₜ > τ**.  
The yearly mask **E_τ** is binary (1 if at least one event occurs in the year).

### Threshold selection (example: τ = 0.26)
**Primary diagnostic (per year)** — **Cliff’s delta (δ)** between CE in **OUT** vs **IN** (non‑parametric effect size; sign > 0 means OUT > IN). We also record the **median difference** `median(OUT) − median(IN)` and **area_frac**.  
**Sensitivity curve** — plot **δ(τ)** per year and the **inter‑annual mean ±1σ**, **scaled to 0–100** for readability.  
**Decision** — choose τ that maximizes separation (lower CE in IN, higher in OUT), with realistic spatial extent and adequate samples. In our experiments, **τ = 0.26** has been effective; adjust if your plots suggest otherwise.

> **AOI usage.** When an **Area of Interest (AOI)** shapefile is provided (e.g., `Elalab_AOI.shp` or `Cafine_AOI.shp`), the **`area_frac`** metric is computed **inside the AOI only**; otherwise, it uses the full image extent.

---

## Key Outputs (per area)
- **Per‑threshold, per‑year rasters**: `<Area>_<Year>_earlyveg_thr_XX.tif`  
- **Per‑threshold, per‑year violin plots**: `Images/<Area>/violins_<Year>/*thr_XX.png`  
- **Per‑threshold CSVs**: `<Area>/<Year>/*_threshold_sensitivity.csv` with `tau`, `n_in`, `n_out`, `area_frac`, `med_diff`, `cliffs_delta`  
- **Final per‑year rasters (chosen τ)**: `<Area>_<Year>_earlyveg_FINAL_thr_XX.tif`  
- **AM overlap (intersection 3/3)**:  
  - Raster: `associated_mangrove_consensus_thr_XX.tif` **and** `associated_mangrove_consensus_thr_XX_clean.tif`  
  - Shapefile: `associated_mangrove_consensus_thr_XX.shp` **and** `associated_mangrove_consensus_thr_XX_clean.shp`  
  - Final violin: `violins_final_thr_XX.png`  
- **Sensitivity figures**: `sensitivity_vs_tau.png` (Cliff’s δ, **0–100 normalized**), `area_vs_tau.png` (area vs τ)  
- **Excel of points**: `AM_points_summary_thr_XX.xlsx` with `ALL`, `IN_AM`, `OUT_AM`

---


## Quick Use
1. **Run** `Cafine_Cafal_detection.ipynb` and/or `Elalab_detection.ipynb` to generate **per‑threshold rasters/plots** per year.  
2. **Inspect** the violin plots and **Cliff’s δ sensitivity curves** and set `CHOSEN_THR` (e.g., `0.26`).  
3. **Re‑run** to build **final per‑year rasters**, the **AM** (raw + `_clean`), the **Excel** tables, and (optionally) the **inline Folium map**.

---

## Dependencies
Python 3.x · `numpy` · `pandas` · `rasterio` · `geopandas` · `matplotlib` · `shapely` · `folium` · `xlsxwriter`

## 👤 Authors
**Jesús Céspedes** and **Jaime Garbanzo‑León** — September 2025
