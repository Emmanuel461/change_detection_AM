#  NDVI-Based Early Vegetation & Salinity Zoning — (2019, 2022, 2023)

## Overview
This repository performs a spatiotemporal **NDVI** analysis to detect the **first appearance of vegetation** (early emergence). The working hypothesis is that areas where vegetation emerges earlier correspond to **low soil salinity** (“sweet zones”), while areas that take longer to show vegetation are **more saline**.  
The workflow runs **per year**, produces **violin plots** (IN vs OUT in **CE**, electrical conductivity), and then applies a **single user-chosen NDVI threshold** to each year to create three final masks and their **overlap** (the **Associated Mangrove / AM** polygon).

**Sensors & bands**
- **2019 (Sentinel-2)**: 2 bands per image — **B4 = Red (index 0)**, **B8 = NIR (index 1)**.
- **2022–2023 (Planet)**: 4 bands per image — **band 3 = Red (index 2)**, **band 4 = NIR (index 3)**.
- Input rasters are named `YYYY_MM_DD.tif` and stacked chronologically.

##  Folder Structure (example with cafine)
```
threshold_method_for_NDVI_change_detection/
├─ Codes/
│  └─ cafine_flow_simple.py
├─ Data/
│  ├─ Images/
│  │  └─ cafine/
│  │     ├─ 2019_sentinel/*.tif
│  │     ├─ 2022/*.tif
│  │     └─ 2023/*.tif
│  └─ Shapefiles/
│     └─ Cafine_salt_UTM.shp        # contains CE (soil electrical conductivity)
├─ Images/
│  └─ cafine/
│     ├─ violins_2019/*.png
│     ├─ violins_2022/*.png
│     ├─ violins_2023/*.png
│     └─ violins_final/violins_final_thr_XX.png
└─ Results/
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

##  Workflow (summary)
1. **Per-year threshold sweep**. For each year (2019, 2022, 2023), evaluate NDVI thresholds:
   <p><strong>&tau; ∈ [0.15, 0.50] (step 0.02)</strong></p>
   At each threshold:
   - Detect <strong>early emergence</strong> using the rule:
     <p><span>&Delta;NDVI = NDVI<sub>t</sub> − NDVI<sub>t−1</sub> &gt; 0.10 and NDVI<sub>t</sub> &gt; &tau;</span></p>
   - Save the <strong>binary raster</strong> (1 = pixel experienced early emergence at least once within the year).
   - Spatially join CE sample points (<code>Cafine_salt_UTM.shp</code>) to the binary raster (IN = inside early emergence zone; OUT = outside) and produce <strong>violin plots</strong> (CE IN vs CE OUT) per threshold and per year.

2. **Manual threshold selection**. After reviewing violin plots and metrics, set <code>CHOSEN_THR</code> (e.g., <strong>0.31</strong>).

3. **Apply the chosen threshold per year**. With <code>CHOSEN_THR</code>, recompute and save <strong>three final rasters</strong>, one per year.

4. **Final overlap (AM)**. <strong>Intersect</strong> the three final masks → the <em>Associated Mangrove (AM)</em> polygon (raster + shapefile). Additionally, generate a <strong>final violin plot</strong> comparing CE <em>inside</em> (IN_AM) vs <em>outside</em> (OUT_AM) the AM.

5. **Tables & map**. Export an <strong>Excel</strong> file (<code>AM_points_summary_thr_XX.xlsx</code>) with sheets <code>ALL</code>, <code>IN_AM</code>, <code>OUT_AM</code>. In a notebook, display a Folium map inline with the AM polygon and colored points.

##  NDVI, Early Emergence & Threshold Selection

### NDVI per date
NDVI at time <em>t</em> is computed as:
<p align="center"><strong>NDVI<sub>t</sub> = (NIR<sub>t</sub> − Red<sub>t</sub>) / (NIR<sub>t</sub> + Red<sub>t</sub>)</strong></p>

### First appearance (per-pixel rule)
Let <span>&Delta;<sub>t</sub>(x) = NDVI<sub>t</sub>(x) − NDVI<sub>t−1</sub>(x)</span>. A pixel <em>x</em> exhibits first appearance at the earliest time <em>t</em> such that:
<p align="center"><span>&Delta;NDVI &gt; 0.10 and NDVI<sub>t</sub> &gt; &tau;</span></p>
The yearly early-emergence mask <span>E<sub>&tau;</sub></span> is binary (1 = at least one first-appearance event within the year).

### Threshold selection (example: τ = 0.31)
<strong>Primary metric (per year)</strong> — difference of <strong>medians</strong> of CE between OUT and IN:
<p align="center"><span>&Delta;<sup>(y)</sup>(&tau;) = median(CE<sub>OUT</sub><sup>(y)</sup>) − median(CE<sub>IN</sub><sup>(y)</sup>)</span></p>
<strong>Aggregate score (across years):</strong>
<p align="center"><span>S(&tau;) = average<sub>y ∈ {2019, 2022, 2023}</sub>{ &Delta;<sup>(y)</sup>(&tau;) }</span></p>
<strong>Constraints and decision:</strong>
- <strong>Area constraint</strong> — discard thresholds with trivial area fractions (e.g., outside 5–60%).  
- <strong>Sample size</strong> — ensure non-negligible counts in IN and OUT.  
- <strong>Final rule</strong> — choose the <span>&tau;</span> that maximizes separation (lower CE in IN, higher in OUT) with reasonable area and samples, and with violin plots showing minimal overlap.  
In this dataset, <strong>&tau; = 0.XXX</strong> emerged as the best value and was applied per year to build the AM.

## Key Outputs
- <strong>Per-threshold, per-year rasters</strong>: <code>cafine_&lt;year&gt;_earlyveg_thr_XX.tif</code>  
- <strong>Per-threshold, per-year violin plots</strong>: <code>Images/cafine/violins_&lt;year&gt;/*thr_XX.png</code>  
- <strong>Final (chosen threshold) per-year rasters</strong>: <code>cafine_&lt;year&gt;_earlyveg_FINAL_thr_XX.tif</code>  
- <strong>AM overlap</strong>:  
  - Raster: <code>associated_mangrove_consensus_thr_XX.tif</code>  
  - Shapefile: <code>associated_mangrove_consensus_thr_XX.shp</code>  
  - Final violin: <code>violins_final_thr_XX.png</code>  
- <strong>Excel of points</strong>: <code>AM_points_summary_thr_XX.xlsx</code> with <code>ALL</code>, <code>IN_AM</code>, <code>OUT_AM</code>.

## Quick Use
1. <strong>Run</strong> <code>Codes/cafine_flow_simple.py</code> (leave <code>CHOSEN_THR = None</code>): generates all per-threshold rasters and violin plots per year.  
2. <strong>Inspect</strong> the violin plots and set <code>CHOSEN_THR</code> (e.g., <code>0.31</code>).  
3. <strong>Re-run</strong> the script: creates the three final per-year rasters, the AM overlap (raster + shapefile), and the final violin.  
4. <strong>(Notebook optional)</strong>: use the provided cell to export the Excel and display the Folium map inline (AM polygon + IN/OUT points).

##  Dependencies
- Python 3.x  
- <code>numpy</code>, <code>pandas</code>, <code>rasterio</code>, <code>geopandas</code>, <code>matplotlib</code>, <code>shapely</code>, <code>folium</code>, <code>xlsxwriter</code>

## Authors
- <strong>Jesús Céspedes</strong> and <strong>Jaime Garbanzo-León</strong>  
- <strong>Date</strong>: August 2025
