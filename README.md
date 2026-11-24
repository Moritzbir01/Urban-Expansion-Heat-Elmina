# Urban expansion and land surface temperature in Elmina

This SDS 320 Spatial Data Analytics project maps candidate urban expansion in Elmina, Klang Valley, Malaysia, and examines its association with relative land surface temperature (LST) change. It compares Landsat 5 TM imagery from **31 May 2004** with Landsat 9 OLI-2/TIRS-2 imagery from **17 May 2025**. Both acquisitions are Landsat Collection 2 Level-2 products sourced through the Microsoft Planetary Computer STAC catalogue.

## Data preparation

The study area spans 101.50–101.54° E and 3.16–3.20° N (WGS 84). The project's local `QA_PIXEL` check reported 0% cloud cover within the study area on both dates under the workflow's QA rules and valid-pixel denominator. The workflow applies product scaling and valid-data masks and checks that both dates share a 30 m grid in UTM zone 47N (EPSG:32647), with matching dimensions (148 × 149) and affine transforms.

## Change detection and spatial validation

| Approach | Method | Reported result |
| --- | --- | --- |
| NDVI/NDBI baseline | Flag candidate urban expansion where ΔNDVI ≤ −0.20 and ΔNDBI ≥ +0.10. Each difference is 2025 minus 2004. | 47.03% of valid baseline pixels met both criteria. |
| Pretrained ChangeStar | Run an exploratory zero-shot transfer experiment through `torchange` and evaluate scores against the manual QGIS reference labels. | ROC-AUC **0.497**, approximately chance-level discrimination. |
| Supervised Siamese CNN | Train on pairs of **9 × 9** multispectral patches with **six bands per date**, using QGIS change and no-change labels. Evaluate with spatial two-fold cross-validation. | Accuracy **0.912**, precision **0.875**, recall **1.000**, F1 **0.933**, ROC-AUC **0.994**. |

The manual QGIS raster provides local change and no-change labels for evaluation and CNN training. Spatial separation between folds reduces dependence between training and evaluation samples. The CNN classification threshold of **0.58** was optimized using the cross-validation results. The thresholded metrics are CV estimates after tuning, and independent held-out performance remains unmeasured. ROC-AUC summarizes discrimination across thresholds.

For full-area mapping, a separate final CNN was fitted on **all 294 labelled patches** and applied at the CV-selected threshold. The performance metrics above describe the fold models. The mapped CNN and NDVI/NDBI masks agreed on **75.0%** of compared pixels, with a change-class **Jaccard index of 0.651**. These figures measure agreement between the maps.

The ChangeStar transfer was unsuccessful for this task. Differences in image characteristics and spatial resolution, along with the mismatch between the pretrained change target and the local urban-expansion labels, offer plausible explanations for the near-chance result.

## Relative land surface temperature

For each mask, the contrast is the mean relative LST change of the changed group minus that of the unchanged group.

| Change mask | Mean relative LST contrast |
| --- | ---: |
| NDVI/NDBI baseline | +4.79 °C |
| Siamese CNN full-area map | +4.04 °C |
| Manual QGIS reference | +6.91 °C |

All three masks show higher mean relative LST change in the changed group. The LST comparison excludes ChangeStar because its scores did not discriminate the local reference classes.

## Project files and reproduction

The repository tracks this README. The notebook and reference GeoTIFF are separate inputs. Use the following local layout.

```text
Elmina_heat_CLEAN.ipynb
qgis_reference/
  elmina_reference_change_mask.tif
```

1. Obtain the clean notebook and the QGIS reference raster. Start Jupyter from the directory above so the notebook can resolve the relative reference path.
2. Set up a Python environment with Jupyter, NumPy, Rasterio, Matplotlib, scikit-learn and PyTorch. The acquisition workflow uses openEO and STAC. The ChangeStar experiment also requires `torchange` and pretrained weights. Consult the notebook's imports and model-loading cell for additional dependencies and checkpoint details.
3. To regenerate imagery inputs, authenticate with an openEO backend that supports `load_stac`. Keep credentials local. Use the acquisitions and bounds above and follow the notebook for scene selection, product scaling, QA and grid checks.
4. Restart the kernel and run all cells in order. Use the notebook for the masking rules, spatial split, threshold selection and relative LST calculation.

## Limitations

- The comparison uses two May acquisitions from different sensors. Weather, atmospheric conditions, soil moisture and sensor differences can influence the observed changes. The LST contrasts describe a spatial association. The study design cannot establish a causal warming effect.
- At 30 m, individual pixels may mix built surfaces, bare soil and vegetation. Vegetation loss and exposed soil can also satisfy the index rule, so its mapped coverage is a screening result.
- Reference interpretation and spatial autocorrelation can affect model performance estimates. The manual labels and two spatial folds cover one study area. Transfer to other areas, dates or sensors remains untested.

## References

- U.S. Geological Survey. [Landsat Collection 2 Level-2 Science Products](https://www.usgs.gov/landsat-missions/landsat-collection-2-level-2-science-products) and [Surface Temperature](https://www.usgs.gov/landsat-missions/landsat-collection-2-surface-temperature).
- Microsoft Planetary Computer. [Landsat Collection 2 Level-2 STAC collection](https://planetarycomputer.microsoft.com/api/stac/v1/collections/landsat-c2-l2).
- Zheng, Z., Ma, A., Zhang, L., and Zhong, Y. (2021). [Change Is Everywhere: Single-Temporal Supervised Object Change Detection in Remote Sensing Imagery](https://openaccess.thecvf.com/content/ICCV2021/papers/Zheng_Change_Is_Everywhere_Single-Temporal_Supervised_Object_Change_Detection_in_Remote_ICCV_2021_paper.pdf). ICCV.
- [Changen2 documentation and pretrained models in torchange](https://github.com/Z-Zheng/pytorch-change-models/tree/main/torchange/models/changen2).
