#  NDVI-Based Early Vegetation & Salinity Zoning — **Elalab** & **Cafine**

This repository implements a reproducible workflow to detect the **first appearance of vegetation** from multi-temporal **NDVI** and use it as a proxy for **soil salinity**. Pixels that green up earlier tend to sit over **low-salinity (“sweet”) soils**, while delayed greening suggests higher salinity. The **same layout and methods** are applied to two study areas: **Elalab** and **Cafine**.

---

##  Sensors & Bands
- **Elalab (2021, 2022, 2024)** — Planet (≥4 bands): **Red = band 3 (index 2)**, **NIR = band 4 (index 3)**.  
- **Cafine (2019, 2022, 2023)** — Sentinel‑2 for **2019** (2 bands): **B4 = Red (index 0)**, **B8 = NIR (index 1)**; Planet for **2022–2023** as above.  
- All rasters are named `YYYY_MM_DD.tif` and processed chronologically.

---

##  Folder Structure (shared)
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
│  │  └─ cafine/
│  │     ├─ 2019_sentinel/*.tif
│  │     ├─ 2022/*.tif
│  │     └─ 2023/*.tif
│  └─ Shapefiles/
│     ├─ Elalab_salt_UTM.shp    # CE (electrical conductivity)
│     └─ Cafine_salt_UTM.shp    # CE (electrical conductivity)
├─ Images/
│  ├─ Elalab/
│  │  ├─ violins_2021/*.png
│  │  ├─ violins_2022/*.png
│  │  ├─ violins_2024/*.png
│  │  └─ violins_final/violins_final_thr_XX.png
│  └─ cafine/
│     ├─ violins_2019/*.png
│     ├─ violins_2022/*.png
│     ├─ violins_2023/*.png
│     └─ violins_final/violins_final_thr_XX.png
└─ Results/
   ├─ Elalab/
   │  ├─ 2021/elalab_2021_earlyveg_thr_XX.tif
   │  ├─ 2022/elalab_2022_earlyveg_thr_XX.tif
   │  ├─ 2024/elalab_2024_earlyveg_thr_XX.tif
   │  ├─ 2021/elalab_2021_earlyveg_FINAL_thr_XX.tif
   │  ├─ 2022/elalab_2022_earlyveg_FINAL_thr_XX.tif
   │  ├─ 2024/elalab_2024_earlyveg_FINAL_thr_XX.tif
   │  ├─ associated_mangrove_consensus_thr_XX.tif
   │  ├─ associated_mangrove_consensus_thr_XX.shp
   │  └─ AM_points_summary_thr_XX.xlsx
   └─ cafine/
      ├─ 2019/cafine_2019_earlyveg_thr_XX.tif
      ├─ 2022/cafine_2022_earlyveg_thr_XX.tif
      ├─ 2023/cafine_2023_earlyveg_thr_XX.tif
      ├─ 2019/cafine_2019_earlyveg_FINAL_thr_XX.tif
      ├─ 2022/cafine_2022_earlyveg_FINAL_thr_XX.tif
      ├─ 2023/cafine_2023_earlyveg_FINAL_thr_XX.tif
      ├─ associated_mangrove_consensus_thr_XX.tif
      ├─ associated_mangrove_consensus_thr_XX.shp
      └─ AM_points_summary_thr_XX.xlsx
```

---

## Workflow (per area)
1. **Per‑year threshold sweep.** For each year in the area, evaluate NDVI thresholds:  
   <p><strong>&tau; ∈ [0.15, 0.50] (step 0.02)</strong></p>
   At each threshold:
   - Detect <strong>early emergence</strong> using:  
     <p><span>&Delta;NDVI = NDVI<sub>t</sub> − NDVI<sub>t−1</sub> &gt; 0.10 and NDVI<sub>t</sub> &gt; &tau;</span></p>
   - Save the <strong>binary raster</strong> (1 = pixel experienced early emergence at least once within the year).
   - Spatially join CE sample points (Elalab/Cafine) to the mask and produce **violin plots** (CE IN vs CE OUT) per threshold and year.

2. **Manual threshold selection (per area).** After reviewing plots/metrics, set <code>CHOSEN_THR</code> (e.g., <strong>0.31</strong>). Use a distinct τ per area if needed.

3. **Apply the chosen threshold per year.** With <code>CHOSEN_THR</code>, recompute and save the **final rasters** for each year in the area.

4. **Final overlap (AM).** **Intersect** the final masks per area to obtain the **Associated Mangrove (AM)** (raster + shapefile). Also generate a **final violin** comparing CE **IN_AM** vs **OUT_AM**.

5. **Tables & map.** Export an **Excel** (`AM_points_summary_thr_XX.xlsx`) with sheets **ALL**, **IN_AM**, **OUT_AM**; in notebooks, display an inline **Folium** map with the AM polygon and colored points.

---

## NDVI, Early Emergence & Threshold Selection

### NDVI per date
NDVI at time <em>t</em> is computed as:  
<p align="center"><strong>NDVI<sub>t</sub> = (NIR<sub>t</sub> − Red<sub>t</sub>) / (NIR<sub>t</sub> + Red<sub>t</sub>)</strong></p>

### First appearance (per‑pixel rule)
Let <span>&Delta;<sub>t</sub>(x) = NDVI<sub>t</sub>(x) − NDVI<sub>t−1</sub>(x)</span>. A pixel <em>x</em> exhibits first appearance at the earliest time <em>t</em> such that:  
<p align="center"><span>&Delta;NDVI &gt; 0.10 and NDVI<sub>t</sub> &gt; &tau;</span></p>
The yearly early‑emergence mask <span>E<sub>&tau;</sub></span> is binary (1 = at least one first‑appearance event within the year).

### Threshold selection (example: τ = 0.31)
<strong>Primary metric (per year)</strong> — difference of <strong>medians</strong> of CE between OUT and IN:  
<p align="center"><span>&Delta;<sup>(y)</sup>(&tau;) = median(CE<sub>OUT</sub><sup>(y)</sup>) − median(CE<sub>IN</sub><sup>(y)</sup>)</span></p>
<strong>Aggregate score (within an area):</strong>  
<p align="center"><span>S(&tau;) = average<sub>y ∈ {years}</sub>{ &Delta;<sup>(y)</sup>(&tau;) }</span></p>
<strong>Decision:</strong> choose the <span>&tau;</span> that maximizes separation (lower CE in IN, higher in OUT) with reasonable spatial realism and sample counts, and with violin plots showing minimal overlap.  
In our experiments, <strong>&tau; = 0.31</strong> has been effective; adjust if your violin plots suggest otherwise.

---

## Key Outputs (per area)
- **Per‑threshold, per‑year rasters**: `<area>_<year>_earlyveg_thr_XX.tif`  
- **Per‑threshold, per‑year violin plots**: `Images/<Area>/violins_<year>/*thr_XX.png`  
- **Final per‑year rasters (chosen τ)**: `<area>_<year>_earlyveg_FINAL_thr_XX.tif`  
- **AM overlap**:  
  - Raster: `associated_mangrove_consensus_thr_XX.tif`  
  - Shapefile: `associated_mangrove_consensus_thr_XX.shp`  
  - Final violin: `violins_final_thr_XX.png`  
- **Excel of points**: `AM_points_summary_thr_XX.xlsx` with `ALL`, `IN_AM`, `OUT_AM`

---

## Quick Use
1. **Run** `elalab_flow.py` and/or `cafine_flow_simple.py` with `CHOSEN_THR = None` to generate **all per‑threshold rasters** and **violin plots** per year.  
2. **Inspect** the violin plots and set `CHOSEN_THR` for each area (e.g., `0.31`).  
3. **Re‑run** to build the final per‑year rasters, the **AM** (raster + shapefile), the **Excel** tables, and (optionally) the **inline Folium map**.

---

## Dependencies
Python 3.x · `numpy` · `pandas` · `rasterio` · `geopandas` · `matplotlib` · `shapely` · `folium` · `xlsxwriter`

## 👤 Authors
**Jesús Céspedes** and **Jaime Garbanzo-León** — August 2025
