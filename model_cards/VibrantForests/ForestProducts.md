# Model card for ForestProducts

Jump to section:

- [Model details](#model-details)
- [Intended use](#intended-use)
- [Factors](#factors)
- [Metrics](#metrics)
- [Evaluation data](#evaluation-data)
- [Training data](#training-data)
- [Quantitative analyses](#quantitative-analyses)
- [Ethical considerations](#ethical-considerations)
- [Caveats and recommendations](#caveats-and-recommendations)
- [Acknowledgements](#acknowledgements)

## Overview
ForestProducts is a multi-target regression model developed by [Vibrant Planet](https://www.vibrantplanet.net/) to allocate aboveground live tree biomass (AGB) into four forest product categories. ForestProducts is integrated into a pipeline used to estimate forest attributes from remote sensing called VibrantForests.

### Contributors
David Diaz, Luke Zachmann, Tony Chang, Nathan Rutenbeck, Vincent Landau, Mike Cartmill, Scott Conway

### Model date
November 2025

### Model version
1.0.0

### Model type
ForestProducts is a series of `HistGradientBoostingRegressor` models combined in a `RegressorChain` using `scikit-learn`. 

### Model details
ForestProducts allocates AGB (provided as an input feature) into four mutually-exclusive forest product categories: sawtimber, pulp, sub-merchantable, and non-merchantable. All four product volumes are predicted in units of metric tons per hectare, and sawtimber is also estimated in units of Scribner boardfoot volume (MBF, thousands of board feet) per hectare.

The definitions of these four forest product categories are derived from specifications used in the US Forest Service's Forest Inventory & Analysis (FIA) program:
* Sawtimber: biomass in the bole of timber species trees with diameter at breast height (DBH) ≥ 9" for softwood (or DBH ≥ 11" for hardwood) timber species from a 1' stump up to a minimum top diameter of 7" for softwood (or 9" for hardwood) timber species.
* Pulp: bole biomass in timber species with DBH ≥ 5" from a 1' stump up to a minimum top diameter of 4", excluding any biomass that qualifies as sawtimber. 
* Submerchantable: woody biomass in tops and branches of all trees; all aboveground woody biomass for all trees with DBH < 5"; and all aboveground woody biomass of woodland (i.e., non-timber/non-commercial) species.  
* Nonmerchantable: biomass in foliage; biomass in stumps of timber species trees with DBH ≥ 5". 

Forest product estimates refer only to standing live biomass. The model does not characterize potentially-merchantable biomass in standing dead trees nor in downed dead wood.

## Intended use

### Primary intended uses
This model was designed to support forest mapping and monitoring with direct applications in a decision support system for wildfire risk assessment and land management planning. It is intended to produce estimates for forest product volumes with annual update frequency at sufficient resolution to enable timber utilization to be effectively incorporated into planning and prioritization activities at spatial scales ranging from individual forest stands up to large landscapes. The model was developed with an initial geographic focus on the contiguous United States (CONUS) following modeling approaches that can be readily extended to other forested regions. 

ForestProducts outputs are also intended to be useful for regional analyses characterizing the potential supply for forest product facilities such as sawmills, pulp mills, etc. and for a growing array of operations that utilize woody biomass to produce energy or create value-added products beyond dimensional lumber and pulp.

### Primary intended users
The intended direct users of ForestProducts outputs are natural resource managers and other professionals leading wildfire risk assessments, community wildfire protection planning, and forest restoration planning through the [Vibrant Planet Platform](https://www.vibrantplanet.net/platform). These use cases generally involve access to zonal summaries of forest and other land attributes at the scale of management units rather than accessing raster data directly. 

We also anticipate uses of ForestProducts outputs in woodshed-scale analyses by and for forest products facilities (e.g., sawmill investors and operators), wood consumers, and policymakers concerned with enhancing and sustaining local market connections and processing capacity, monitoring wood utilization across landscapes over time, and enabling the expansion of forest restoration, resilience, and wildfire risk reduction activities. 

### Out-of-scope use cases
ForestProducts predictions are made on an areal basis at 10m resolution, and do not support merchantability calculations for individual trees.

Additionally, the model was not developed to apply variable merchantability specifications or defect estimates that may be necessary for grading wood for specialty or export use cases. 

Finally, this model allocates aboveground live tree biomass into four product categories. It was not designed to allow confident quantification of wood quality nor volume that might be generated from salvage logging in stands affected by wildfire, pests, or other natural disturbances where a significant proportion of tree mortality has occurred.

## Factors
The key factors shaping the development and application of ForestProducts include reliance upon a forest structure model to generate the necessary forest attributes in wall-to-wall maps, and the definitions of forest product categories derived from the FIA program.  

Biogeographic factors that influence tree form and the allocation of AGB into different product categories were incorporated by including forest type and ecoregion as key model inputs.

## Metrics
We evaluated model performance at plot scale using National Forest Inventory data. Model performance is reported using Mean Absolute Error (MAE), Root Mean Squared Error (RMSE), mean bias, R-squared (R²), and Pearson's R. Qualitative assessments were conducted through visual inspections of scatterplots of predicted versus observed values. 

## Evaluation data
### Datasets
We utilized inventory data from publicly-available FIA databases for both model training and validation. These inventory data were coupled with a global delineation of ecoregions published [Dinerstein, et al. (2017)](https://doi.org/10.1093/biosci/bix014).

### Motivation
The FIA program provides a systematic and spatially balanced inventory of forests across CONUS. The geographic expanse and diversity, sample size, and revisit frequency provide a robust data source for model development and validation. 

### Preprocessing
We extracted attributes from the FIA database including canopy cover, forest type group, canopy height, basal area, aboveground biomass, and quadratic mean diameter. The ecoregion for each sample was defined using a spatial join of the fuzzed FIA plot coordinates with the global ecoregions layer.

## Training data
FIA inventory data were split into train (80%) and test (20%) partitions using a stratified sampling approach to retain the same balance of samples by forest type group and ecoregion in each partition.

## Data used at inference time
At inference time, we relied upon the VibrantForests model to produce wall-to-wall estimates of AGB, basal area, quadratic mean diameter, canopy cover, and canopy height. These forest structure estimates were coupled with the ecoregions data layer and wall-to-wall estimates of Forest Type Groups from [FIA Bigmap](https://www.arcgis.com/home/item.html?id=4f78c504917a4f35b1c3191e94b5d565).

## Quantitative analyses
Performance evaluation across all forest type groups and ecoregions demonstrated precise estimates of forest product volumes in all categories, with minimal bias and high levels of correlation between observations and predictions for all product categories (Pearson R values from 0.78 to 0.98), and the vast majority of variance among samples explained for the sawlog, submerch, and nonmerch categories (R² values ranging from 0.88 to 0.95). A lower level of explained variance was observed for the pulp target (R²=0.31), which we believe to be related to the calculation of pulp biomass as the remainder of merchantable bolewood after subtracting sawlog biomass.

| Target                    |   MAE |   RMSE |   Mean Bias |   R² |   Pearson R | Obs (Mean ± SD)   | Pred (Mean ± SD)   |
|:--------------------------|------:|-------:|------------:|-----:|------------:|:------------------|:-------------------|
| Sawlog Biomass (Mg/ha)    |  10.1 |   19.3 |       +0.47 | 0.95 |        0.98 | 46.2 ± 85.4       | 46.3 ± 87.3        |
| Pulp Biomass (Mg/ha)      |   9.4 |   17.8 |       -0.68 | 0.31 |        0.78 | 25.4 ± 28.4       | 25.4 ± 28.3        |
| Submerch Biomass (Mg/ha)  |   6.4 |   11.2 |       -0.35 | 0.88 |        0.95 | 38.2 ± 34.9       | 38.2 ± 34.8        |
| Nonmerch Biomass (Mg/ha)  |   2.5 |    4.9 |       +0.55 | 0.95 |        0.98 | 20.4 ± 22.0       | 20.5 ± 22.3        |
| Boardfoot Volume (MBF/ha) |   5.1 |   13.0 |       +0.14 | 0.91 |        0.96 | 20.0 ± 43.3       | 20.1 ± 44.7        |

| ![Overall performance evaluation of ForestProducts](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantForests/ForestProducts/1.0.0/overall_performance.png) |
| :-- |
| The graphics above display 2-D histograms of observed and predicted forest product volumes using a heatmap, with higher densities of samples displayed in red and lower densities of samples in dark blue. The 1:1 line indicating perfect correspondence between predictions and observations is overlaid diagonally in each graphic as a red dashed line. |

## Ethical considerations
The data used and generated by ForestProducts are not considered sensitive nor to pose substantial risks to human health or safety. The data are intended to be instrumental in decision-making that may indirectly enable human health and safety to be better protected through more cost-effective and targeted wildfire and forest restoration planning. Similar data sources already exist at lower resolution and periodic update frequency produced by the US Forest Service (e.g., [TreeMap by Riley et al. 2021](https://www.nature.com/articles/s41597-020-00782-x)), and more precise maps of forest product volumes are commonly generated in public and private sectors based on lidar data.

We are conscious of the fact that forest protection and conservation communities in the USA and abroad are concerned about exploitation of forests in a manner that is myopically focused on timber utilization at the expense of environmental and social costs. Given the preexisting availability of similar forest product data within the timber industry, by state natural resource agencies, and at the US Forest Service, we do not believe the provision of forest product data from publicly-available satellite imagery on a regular cadence is going to meaningfully alter the behavior of public or private land managers in terms of increasing or decreasing harvest levels. We are hopeful that more consistent data on forest product volumes allows the judicious management of wood resources to be incorporated alongside the variety of other forest and community outcomes that can be evaluated and acted upon through the decision support systems we develop. 

## Caveats and recommendations
This model was developed and evaluated with an initial focus on the contiguous USA. Caution is advised against naive application beyond that scope without additional effort to update the training and evaluation datasets to determine model performance in other regions.

Although there remains significant interest (and debate) surrounding salvage logging of dead or downed trees following natural disturbances, this model has not been trained to estimate the merchantability of dead trees, and is unlikely to provide accurate estimates of merchantable volume in the wake of acute mortality events like wildfire, extreme wind, or severe pest outbreaks. 

## Acknowledgements
The development of ForestProducts was supported by funding from a grant by the Doris Duke Foundation to American Forests for the development of the Forest Innovation Platform.

This model card format was adapted from a Markdown template developed by [Christian Garbin](https://github.com/fau-masters-collected-works-cgarbin/model-card-template), based on sections and prompts from [Mitchell et al. (2019)](https://arxiv.org/abs/1810.03993).
