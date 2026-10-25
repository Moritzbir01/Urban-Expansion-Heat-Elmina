# Heat-Klang-Valley – SDS320

An exploratory remote sensing project investigating urban expansion along the Elmina expansion corridor in Klang Valley, Malaysia. The aim is to identify changes in vegetation and built-up areas and investigate their relationship with land surface temperature.

## Current status (25 October 2026)

The Jupyter workflow now includes bi-temporal Landsat change detection, NDVI/NDBI differences, a preliminary candidate-urbanisation mask, and absolute and relative land surface temperature (LST) comparisons. These are exploratory results; the change mask has not yet been validated against reference data.

| Acquisition date | Satellite | RGB bands (red, green, blue) |
| --- | --- | --- |
| 31 May 2004 | Landsat 5 | `TM_B3`, `TM_B2`, `TM_B1` |
| 17 May 2025 | Landsat 9 | `OLI_B4`, `OLI_B3`, `OLI_B2` |

The comparison is **2004-05-31 vs 2025-05-17**. Both acquisitions fall in May, which helps limit seasonal differences, but weather, acquisition conditions, and sensor differences remain relevant.

## Study area

Elmina subset in geographic coordinates (WGS84, EPSG:4326):

```python
bbox_list = [101.50, 3.16, 101.54, 3.20]
bbox_stac = {
    "west": 101.50,
    "south": 3.16,
    "east": 101.54,
    "north": 3.20,
}
```

`bbox_list` is used for the STAC search and local cloud check; `bbox_stac` is the dictionary passed to `connection.load_stac(spatial_extent=...)`.

The analysis rasters have the same CRS (**EPSG:32647**), **30 m** resolution, **148 × 149** grid, and matching affine transform, enabling pixel-wise comparison.

## Data and cloud check

Data source: Landsat Collection 2 Level-2 through the [Microsoft Planetary Computer STAC catalogue](https://planetarycomputer.microsoft.com/api/stac/v1/collections/landsat-c2-l2).

The `QA_PIXEL` checks reported **0% cloud cover within the ROI for both dates**. This refers to the local ROI, not the entire scene, and depends on the QA bits, invalid-pixel handling, and denominator used. Values here are reported from the project workflow, not independently recalculated for this README.

## Jupyter workflow

Requirements include a Python/Jupyter environment, access to an openEO backend supporting `load_stac`, and an authenticated openEO connection named `connection`. The specific backend and authentication procedure should be documented in the project notebook.

The RGB step requires `openeo`, `rasterio`, `numpy`, and `matplotlib`; additional packages may be needed for the STAC search and ROI cloud check.

1. Connect to the openEO backend and authenticate.
2. Search for Landsat scenes through STAC and check `QA_PIXEL` within the ROI.
3. Load the two selected acquisitions using the same spatial extent.
4. Export the RGB data as GeoTIFFs and display both images side by side in Jupyter.
5. Load the spectral and thermal products, apply product scaling and valid-data masks, and check spatial alignment.
6. Calculate NDVI, NDBI, their differences (2025 minus 2004), and the preliminary candidate-urbanisation mask.
7. Compare absolute and relative LST and visualise the exploratory results.

The RGB loading and export block is:

```python
landsat_url = (
    "https://planetarycomputer.microsoft.com/api/stac/v1/"
    "collections/landsat-c2-l2"
)

old_final = connection.load_stac(
    landsat_url,
    spatial_extent=bbox_stac,
    temporal_extent=["2004-05-31", "2004-06-01"],
    bands=["TM_B3", "TM_B2", "TM_B1"],
    properties={"platform": lambda x: x == "landsat-5"},
)

new_final = connection.load_stac(
    landsat_url,
    spatial_extent=bbox_stac,
    temporal_extent=["2025-05-17", "2025-05-18"],
    bands=["OLI_B4", "OLI_B3", "OLI_B2"],
    properties={"platform": lambda x: x == "landsat-9"},
)

old_final.execute_batch("elmina_2004_05_31_rgb.tif", out_format="GTiff")
new_final.execute_batch("elmina_2025_05_17_rgb.tif", out_format="GTiff")
```

These filenames specify the RGB exports; the block alone does not reproduce the index or LST analyses. Creating a data cube does not confirm successful backend execution.

For visualisation, the three RGB bands are read with Rasterio and displayed side by side with Matplotlib. The contrast stretch uses the 2nd and 98th percentiles separately for each image. Separate stretches are for display and do not support direct quantitative comparisons of colour or brightness.

### Preliminary change-detection and LST results

| Mean over valid pixels | 2004-05-31 | 2025-05-17 |
| --- | ---: | ---: |
| NDVI | 0.59458 | 0.40187 |
| NDBI | -0.21017 | -0.06159 |
| LST (°C) | 36.77 | 47.81 |

The preliminary rule flags pixels where **ΔNDVI ≤ -0.20 and ΔNDBI ≥ +0.10**, with Δ defined as 2025 minus 2004. **47.03% of valid pixels** meet both conditions and are labelled **candidate urbanisation**. This is a threshold-based screening result, **not a final urbanised-area estimate**; vegetation loss, bare soil, and other changes may also meet the rule.

The raw LST difference must not be interpreted as a causal effect of urbanisation: weather, acquisition conditions, sensor differences, and other factors can influence the two-date comparison.

Mean relative LST change is **+2.23°C** in candidate-urbanisation pixels and **-2.56°C** in other areas, a group difference of approximately **4.79°C**. These relative results describe a spatial association, not validated change labels or causal attribution. The exact relative-LST baseline and valid-pixel masks must be recorded in the notebook for reproduction. The relative-change map uses a diverging colour scale centred on zero; NoData areas are excluded from interpretation.

## Next steps

- Create a reference change mask in QGIS and validate the candidate-urbanisation mask against it, documenting reference sources and labelling criteria.
- Assess classification errors and threshold sensitivity before estimating urbanised area or drawing stronger conclusions about LST.
- Consider a later comparison with pretrained ChangeStar using the same study area and reference mask; no model validation has been completed yet.

## Reproducibility

This README summarises the reported project status. Full reproduction requires the current notebook, backend configuration without credentials, package versions, STAC item IDs, QA bit definitions, product bands and scaling, NoData/valid-pixel masks and denominators, grid/transform checks, threshold implementation, and the exact relative-LST definition. Reference-mask provenance and validation procedures should be added when available. Credentials and tokens must not be published.
