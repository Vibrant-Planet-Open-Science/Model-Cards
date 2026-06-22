# VibrantForests for the Forest Innovation Platform: Model Card<sup>1</sup>

This is a short-form, delivery-specific model card documenting **only the three raster
layers** that Vibrant Planet delivered to the [Forest Innovation Platform](https://forestinnovationplatform.org/)
(FIP), a web application under development by American Forests:

| FIP layer | Units | Produced by | Full model card |
| :-- | :-- | :-- | :-- |
| Relative Stand Density Index (rSDI) | % | ForestDensity v1.0.0 | [Model Card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestDensity-v1.0.0/model_cards/VibrantForests/ForestDensity.md) |
| Basal Area (BA) | m² / ha | ForestStructure v1.1.1 | [Model Card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestStructure-v1.1.1/model_cards/VibrantForests/ForestStructure.md) |
| Merchantable Timber Biomass | metric tons / ha (Mg/ha) | ForestProducts v1.1.0 | [Model Card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestProducts-v1.0.0/model_cards/VibrantForests/ForestProducts.md) |

These layers are produced by **VibrantForests**, which is not a single model but a *pipeline
of models* in which baseline forest-structure estimates feed downstream models that derive
additional attributes. Each of the three FIP layers comes from a different model in that
pipeline, and each model produces several attributes beyond the one delivered to FIP. This
card scopes the documentation down to exactly what FIP received and, in particular, makes
explicit that **Merchantable Timber Biomass is the sum of two separately-modeled product
categories (sawtimber + pulp)**. For complete methodology,
evaluation, and caveats for any layer, follow the link to its full model card above.

Jump to section:

- [Overview](#overview)
- [Delivered products](#delivered-products)
  - [1. Relative Stand Density Index (rSDI)](#1-relative-stand-density-index-rsdi)
  - [2. Basal Area (BA)](#2-basal-area-ba)
  - [3. Merchantable Timber Biomass](#3-merchantable-timber-biomass)
- [Intended use](#intended-use)
- [Data and processing](#data-and-processing)
- [Ethical considerations](#ethical-considerations)
- [Caveats and recommendations](#caveats-and-recommendations)
- [Acknowledgements](#acknowledgements)

## Overview

VibrantForests estimates forest attributes from satellite remote sensing across the contiguous United States (CONUS). For the FIP delivery, three layers were produced from 2024 imagery, mosaicked and tiled at **30 m resolution in EPSG:5070 (CONUS Albers)**, and delivered as cloud-optimized GeoTIFFs (COGs) and tiles. The native resolution of the underlying models is 10 m; the delivered products were resampled from 10 m to 30 m (`AVERAGE`), reprojected from EPSG:6931 to EPSG:5070, and retiled to the CONUS 15 km grid.

### Contributors
David Diaz, Luke Zachmann, Tony Chang, Nathan Rutenbeck, Vincent Landau, Mike Cartmill, Scott Conway, Katharyn Duffy, Andreas Gros, Kiarie Ndegwa, Guy Bayes

### Model date
November 2025 (component models); FIP delivery prepared from 2024 imagery.

### Model version
FIP delivery 1.0.0, composed of: ForestStructure 1.1.1, ForestDensity 1.0.0, ForestProducts 1.1.0.

### Model type
A pipeline of three models of different types, described here in widely-recognized terms:
* **ForestStructure** — a **deep-learning neural network**: a Feature Pyramid Network (FPN)
  built on VibrantMAE, a Vision Transformer-based Masked AutoEncoder. Produces Basal Area.
  ([ForestStructure model card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestStructure-v1.1.1/model_cards/VibrantForests/ForestStructure.md).)
* **ForestDensity** — a **linear mixed-effects quantile regression**, estimated in a Bayesian hierarchical framework. It fits the *maximum size–density relationship* (MSDR) — the classic maximum size–density (self-thinning) boundary in the tradition of Reineke's Stand Density Index — as the upper (90<sup>th</sup>-percentile) envelope of stem density versus average tree size, with **fixed effects** for the overall relationship plus **random effects** that let the line vary by forest type and ecoregion. Produces rSDI. ([ForestDensity model card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestDensity-v1.0.0/model_cards/VibrantForests/ForestDensity.md).)
* **ForestProducts** — a **non-parametric, tree-based ensemble**: gradient-boosted regression trees (`HistGradientBoostingRegressor`, in the same family as Random Forest and XGBoost), wrapped in a `RegressorChain` so the four products are predicted in sequence (each prediction becomes an input feature for the next). Produces the sawtimber and pulp components that are summed into Merchantable Timber Biomass. ([ForestProducts model card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestProducts-v1.0.0/model_cards/VibrantForests/ForestProducts.md).)

## Delivered products

### 1. Relative Stand Density Index (rSDI)

#### What it is
rSDI expresses the stocking of a forest relative to its maximum potential stocking at a point in time (i.e., SDI<sub>t</sub> / SDI<sub>max</sub>), reported as a percentage. In the full [ForestDensity model card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestDensity-v1.0.0/model_cards/VibrantForests/ForestDensity.md) this attribute is referred to as **Relative Density (RD)**; the FIP layer name "rSDI" and "RD" refer to the same quantity.

#### How it is produced
ForestDensity fits Maximum Size-Density Relationship (MSDR) lines — the upper-boundary (self-thinning) limit on how many trees of a given average size a site can support, in the tradition of Reineke's Stand Density Index — and from them SDI<sub>max</sub>. Statistically it is a linear mixed-effects quantile regression: it predicts the 90<sup>th</sup> percentile of stem density as a function of average tree size, forest composition, and region, with random effects that let the relationship vary by forest type, ecoregion, and forest-type-within-ecoregion. At inference, current SDI is derived from ForestStructure's basal area and quadratic mean diameter (QMD) predictions, composition is taken from the FIA Bigmap Forest Type Group layer, and RD = SDI / SDI<sub>max</sub>. A circular smoothing kernel (40 m radius) is applied to SDI<sub>max</sub> and RD, and RD is clamped to 0–200. See the [ForestDensity model card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestDensity-v1.0.0/model_cards/VibrantForests/ForestDensity.md) for full methodology.

#### Performance
rSDI was evaluated against two FIA-derived references: the All-Live-Tree Stocking Percent (ALSTK), and the empirical 90<sup>th</sup> percentile of observed SDI among compositionally-pure plots in each ecoregion. We found strong correspondence between our estimate of RD and FIA's ALSTK, and tight correspondence with the empirical SDI percentile for forest type / ecoregion combinations with substantial sample sizes (rarer combinations drift above the 1:1 line due to the model's hierarchical regularization).

Because ForestDensity's RD/rSDI corresponds closely to FIA's All-Live-Tree Stocking Percent (ALSTK, truncated at 120%), the ALSTK stocking classes offer a familiar interpretive scale for rSDI values:

| ALSTK range | Stocking class |
| :-- | :-- |
| > 100% | Overstocked |
| 60–100% | Fully Stocked |
| 35–60% | Medium Stocked |
| 10–35% | Poorly Stocked |
| < 10% | Nonstocked |

| ![CONUS Relative Density Map](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantForests/ForestDensity/1.0.0/rsdi_conus_map.png) |
| :-- |
| *CONUS Relative Density mapped from VibrantForests' SDI (2024 imagery) and ForestDensity's SDI<sub>max</sub>. Legend labels — LDM = Low Density Management; ZODM = Zone of Optimal Density Management; ZICM = Zone of Imminent Competition Mortality; DDM = Density-Dependent Mortality — adapted from [Ray et al. (2023)](https://research.fs.usda.gov/treesearch/67075) and [Zukswert and Kenefic (2025)](https://research.fs.usda.gov/treesearch/68854).* Note that the ranges visualized in this graphic differ from the `ALSTK` categories. |

| ![Modeled estimates versus FIA observations](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantForests/ForestDensity/1.0.0/estimates_vs_fia.png) |
| :-- |
| *Left: count of FIA plots in hexagonal bins spanning 1 percentage point of ALSTK and RD. Right: correspondence of observed and estimated SDI quantiles, with point size scaled by sample count for each Forest Type Group in each ecoregion.* |

#### Layer-specific caveats
MSDR lines target the 90<sup>th</sup> percentile of field observations, so observations and predictions above SDI<sub>max</sub> are expected. The precision of rSDI is not sufficient to interpret small (single-percentage-point) changes; we recommend using ordinal density/management "zones" rather than raw percentages. Estimates may be conservative (biased low) for stands of large trees (the "Mature Stand Boundary" effect), can show abrupt steps at ecoregion boundaries, and may exhibit ring artifacts around tall buildings because land-cover masking is applied after the smoothing kernel.

### 2. Basal Area (BA)

#### What it is
Basal area is the cross-sectional area of live tree stems per unit of ground area, reported in m² / ha. It is one of five fundamental forest-structure attributes produced by ForestStructure (alongside AGB, QMD, canopy cover, and canopy height); **only BA was delivered to FIP.**

#### How it is produced
ForestStructure is an FPN built on the frozen VibrantMAE encoder. It takes Sentinel-2 June–July–August (JJA) composites as input and predicts forest-structure attributes at 10 m resolution using a 256 × 256 context window, with sliding-window inference mosaicked to seamless rasters. Training targets were generated by applying an allometric model to lidar-derived rasters (USGS 3DEP, Quality Level 2 or better). See the [ForestStructure model card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestStructure-v1.1.1/model_cards/VibrantForests/ForestStructure.md) for full methodology.

#### Performance
BA was among the better-performing structure attributes, with modest saturation appearing at the high end of the observed range.

ForestStructure predicts five attributes jointly; **only BA was delivered to FIP** and results presented here are limited to BA alone.

*Plot-scale (2024 predictions vs. 2010–2018 field plots):*

| Target | MAE | RMSE | Mean Bias | R² | Pearson R | Obs (Mean ± SD) | Pred (Mean ± SD) |
| :-- | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| **BA (m²/ha)** | **19.9** | **27.4** | **+1.8** | **0.24** | **0.57** | **44 ± 32** | **46 ± 27** |

*Region-scale (64,000 ha hexagons, FIA 2012–2022):*

| Target | MAE | RMSE | Mean Bias | R² | Pearson R | Obs (Mean ± SD) | Pred (Mean ± SD) |
| :-- | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| **BA (m²/ha)** | **2.8** | **4.5** | **−0.54** | **0.77** | **0.89** | **8.0 ± 9.4** | **7.4 ± 7.2** |

*BA plot-scale R² is depressed by the time lag between field measurement (2010–2018) and prediction (2024) and by plot-level noise; region-scale statistics are computed on an all-area basis and are influenced by the large number of non-forest (near-zero) pixels. See the [ForestStructure model card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestStructure-v1.1.1/model_cards/VibrantForests/ForestStructure.md) for pixel-scale results and full discussion.*

#### Layer-specific caveats
BA is predicted on an areal basis at 10 m and is not suitable for individual-tree attribution. Predictions reflect 2024 conditions; field comparisons include disturbance-driven mismatches. Built-up, water, and agricultural pixels are masked (zeroed) when ForestStructure outputs are prepared for downstream use.

### 3. Merchantable Timber Biomass

> **Merchantable Timber Biomass = Sawtimber + Pulp** (both in Mg/ha).
>
> The FIP layer is the **sum of two of the four mutually-exclusive forest-product categories**
> that ForestProducts predicts. It **excludes** the submerchantable and nonmerchantable
> categories, and it represents **standing live biomass only** (no standing-dead or downed
> dead wood).

#### What it is
ForestProducts allocates aboveground live tree biomass (AGB) into four mutually-exclusive categories, each in metric tons per hectare. The FIP Merchantable Timber Biomass layer sums the two merchantable categories:

* **Sawtimber** — bole biomass of timber species with DBH ≥ 9" (softwood) or ≥ 11" (hardwood), from a 1' stump to a minimum top diameter of 7" (softwood) or 9" (hardwood).
* **Pulp** — bole biomass of timber species with DBH ≥ 5" from a 1' stump to a 4" minimum top, excluding any biomass that qualifies as sawtimber.

The two categories *not* included in the FIP layer are *submerchantable* (tops/branches, all biomass of trees with DBH < 5", and all biomass of woodland/non-timber species) and *nonmerchantable* (foliage, and stumps of timber species with DBH ≥ 5").

#### How it is produced
ForestProducts is a `RegressorChain` of `HistGradientBoostingRegressor` models. It takes AGB (and other structure attributes from ForestStructure), Forest Type Group (FIA Bigmap), and ecoregion as inputs, and allocates AGB into the four product categories (the four are proportionally scaled to sum to AGB). The FIP delivery was produced with ForestProducts **v1.1.0** (the model version pinned in Vibrant Planet's FIP production pipeline). To recover the delivered layer:

```
Merchantable Timber Biomass (Mg/ha) = Sawtimber (Mg/ha) + Pulp (Mg/ha)
```

#### Performance
Because Merchantable Timber Biomass is delivered as a single combined layer,
we report performance here as a single quantity: for each FIA subplot in the held-out test partition (n = 134,972), the modeled sawtimber and pulp predictions are summed and compared to the summed observed values, using the delivered model (ForestProducts v1.1.0)<sup>2</sup>. The other rows — the sawtimber and pulp components, plus the submerchantable, nonmerchantable, and boardfoot categories that ForestProducts also predicts — are reproduced from the [ForestProducts model card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestProducts-v1.0.0/model_cards/VibrantForests/ForestProducts.md) for context.

| Quantity | Units | MAE | RMSE | Mean Bias | R² | Pearson R | Obs (Mean ± SD) | Pred (Mean ± SD) |
| :-- | :-- | :--: | :--: | :--: | :--: | :--: | :-- | :-- |
| **Merchantable Timber Biomass (Sawtimber + Pulp)** | Mg/ha | **5.9** | **10.3** | **−0.2** | **0.99** | **0.99** | **71.7 ± 94.5** | **71.5 ± 93.3** |
| &nbsp;&nbsp;↳ Sawtimber *(delivered as component of sum)* | Mg/ha | 10.1 | 19.3 | +0.47 | 0.95 | 0.98 | 46.2 ± 85.4 | 46.3 ± 87.3 |
| &nbsp;&nbsp;↳ Pulp *(delivered as component of sum)* | Mg/ha | 9.4 | 17.8 | −0.68 | 0.31 | 0.78 | 25.4 ± 28.4 | 25.4 ± 28.3 |
| Submerchantable *(not delivered)* | Mg/ha | 6.4 | 11.2 | −0.35 | 0.88 | 0.95 | 38.2 ± 34.9 | 38.2 ± 34.8 |
| Nonmerchantable *(not delivered)* | Mg/ha | 2.5 | 4.9 | +0.55 | 0.95 | 0.98 | 20.4 ± 22.0 | 20.5 ± 22.3 |
| Sawtimber boardfoot volume *(not delivered)* | MBF/ha | 5.1 | 13.0 | +0.14 | 0.91 | 0.96 | 20.0 ± 43.3 | 20.1 ± 44.7 |

*The combined Merchantable Timber Biomass — the form in which the layer was delivered to FIP — is estimated with very high precision (R² = 0.99, Pearson R = 0.99) and near-zero mean bias (−0.2 Mg/ha). Its sawtimber and pulp components are individually noisier; pulp in particular shows a lower R² because it is computed as the remainder of merchantable bolewood after subtracting sawtimber. Their offsetting errors largely cancel when summed back into total merchantable volume, so the delivered layer tracks observed merchantable bolewood closely both on average and per plot. The submerchantable, nonmerchantable, and boardfoot rows are the remaining ForestProducts outputs (reproduced from the [ForestProducts model card](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestProducts-v1.0.0/model_cards/VibrantForests/ForestProducts.md)); they were **not** delivered to FIP and are shown only to document the full AGB allocation.*

It is important to note that this evaluation is based on inputs of field-measured canopy height, cover, basal area, and quadratic mean diameter. When applied for wall-to-wall maps, the inputs used by the ForestProducts model are predicted by the ForestStructure model, and error in those predictions may propagate into estimates of Merchantable Timber Biomass.

#### Layer-specific caveats
Estimates are for standing live trees only and are not reliable for quantifying salvage volume after wildfire, wind, or pest-driven mortality. Predictions are areal (10 m) and do not support per-tree merchantability or specialty/export grading.

## Intended use

#### **Primary intended uses** 
These layers support forest mapping and monitoring with annual update frequency at resolutions enabling planning and prioritization from individual stands up to large landscapes. For FIP specifically, they support rapid assessments of forest vulnerability and climate adaptation planning. They were developed with an initial geographic focus on CONUS.

#### **Primary intended users** 
Natural resource managers and analysts leading wildfire risk assessment, community wildfire protection planning, and forest restoration planning — including FIP users at American Forests, users of the [Vibrant Planet Platform](https://www.vibrantplanet.net/platform), and researchers performing regional analyses. These use cases generally involve zonal summaries at the scale of management units rather than direct use of the 10 m / 30 m rasters.

#### **Out-of-scope use cases** 
None of these layers is intended for individual-tree detection or attribution — the resolution is chosen to represent forest attributes on an areal basis. Merchantable Timber Biomass is not intended for salvage estimation or specialty grading. We caution against naive application outside CONUS without additional training and evaluation.

## Data and processing

#### **Inputs** 
Sentinel-2 JJA composites (ForestStructure); US Forest Inventory & Analysis (FIA) data for training and evaluation (all three models); Resolve Ecoregions 2017 ([Dinerstein et al. 2017](https://doi.org/10.1093/biosci/bix014)); the [FIA Bigmap Forest Type Group](https://www.arcgis.com/home/item.html?id=4f78c504917a4f35b1c3191e94b5d565) layer (ForestDensity and ForestProducts inference); and USGS 3DEP lidar (training targets).

#### **FIP delivery processing** 
The 10 m model outputs were combined and masked, then resampled to 30 m (`AVERAGE`), reprojected from EPSG:6931 to EPSG:5070 (CONUS Albers), and retiled to the CONUS 15 km grid for delivery as COGs and tiles. A forest/land-cover mask (zeroing built-up, water, and agricultural pixels, using a mixture of Annual NLCD, Sentinel-2 NDVI, and Overture Maps buildings/roads) is applied so that predictions pass through only over vegetated cover types.

## Ethical considerations

The data used and produced are not sensitive and are not expected to pose substantial risks to human health or safety; they are intended to support more cost-effective and targeted wildfire and forest-restoration planning. Increased availability and precision of data on the volume and potential value of forest products could, in principle, contribute to land speculation or information asymmetry. Given that comparable data already exist at high and moderate resolution from state and federal agencies (e.g., the US Forest Service), research groups, and the private sector, we do not expect these layers to create new or additional risks of harm.

## Caveats and recommendations

These layers were developed and evaluated with an initial focus on CONUS, and the reliance on Sentinel-2 imagery limits applicability prior to 2018. The models are not conditioned to produce temporally-smooth predictions, so time-series use may show non-smooth trends. Each delivered layer carries its own caveats — see the layer-specific notes above and the full model cards: [ForestDensity](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestDensity-v1.0.0/model_cards/VibrantForests/ForestDensity.md), [ForestStructure](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestStructure-v1.1.1/model_cards/VibrantForests/ForestStructure.md), and [ForestProducts](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestProducts-v1.0.0/model_cards/VibrantForests/ForestProducts.md).

## Acknowledgements

The development of these models was supported by funding from a grant by the Doris Duke Foundation to American Forests for the development of the Forest Innovation Platform.

This material is also based upon work supported by the U.S. Department of Agriculture, under agreement number NR233A750004G042, and by the USDA Forest Service, under agreement number #24-CA-11132544-064. Any opinions, findings, conclusions, or recommendations expressed in this publication are those of the author(s) and do not necessarily reflect the views of the U.S. Department of Agriculture. The ForestStructure model builds on a large collection of lidar data acquired and processed through contributions by many colleagues.

<sup>1</sup>: This short-form model card consolidates content from the
[ForestStructure](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestStructure-v1.1.1/model_cards/VibrantForests/ForestStructure.md), [ForestDensity](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestDensity-v1.0.0/model_cards/VibrantForests/ForestDensity.md), and [ForestProducts](https://github.com/Vibrant-Planet-Open-Science/Model-Cards/blob/VibrantForests-ForestProducts-v1.0.0/model_cards/VibrantForests/ForestProducts.md) model cards. Its format follows the sections and prompts from [Mitchell et al. (2019)](https://arxiv.org/abs/1810.03993), adapted from a Markdown template by [Christian Garbin](https://github.com/fau-masters-collected-works-cgarbin/model-card-template).

<sup>2</sup>: The combined Merchantable Timber Biomass metric was generated specifically for this delivery by reproducing the ForestProducts train/test split (stratified by Forest Type Group × ecoregion, 20% test, `random_state=42`), running the production model (`s3://vp-models/ml/prod/forest_products/1.1.0/model.joblib`) on the held-out test partition, summing the per-subplot sawtimber and pulp predictions (and the corresponding observed values), and computing standard regression metrics on the resulting total. Sawtimber, pulp, submerchantable, and nonmerchantable predictions are proportionally scaled to sum to the AGB input, so the sum of the sawtimber and pulp components is the merchantable bolewood total.