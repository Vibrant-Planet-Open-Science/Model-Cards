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
ForestProducts is a neural network developed by [Vibrant Planet](https://www.vibrantplanet.net/) to allocate aboveground live tree biomass (AGB) into four forest product categories and to estimate sawtimber boardfoot volume. ForestProducts is integrated into a pipeline used to estimate forest attributes from remote sensing called VibrantForests.

### Contributors
David Diaz, Luke Zachmann, Tony Chang, Nathan Rutenbeck, Vincent Landau, Mike Cartmill, Scott Conway

### Model date
October 2026

### Model version
2.0.0

### Model type
ForestProducts is a multilayer perceptron (MLP) built in PyTorch and exported to ONNX for inference. It has two hidden layers (128 and 64 units wide) and a learned 8-dimensional embedding for each of its two categorical inputs, forest type group and ecoregion.

### Model details
ForestProducts allocates AGB (provided as an input feature) into four mutually-exclusive forest product categories: sawtimber, pulp, sub-merchantable, and non-merchantable. All four products are estimated in units of metric tons per hectare, and sawtimber is also estimated as boardfoot volume (MBF, thousands of board feet) per hectare.

The definitions of these four forest product categories are derived from the biomass components reported by the US Forest Service's Forest Inventory & Analysis (FIA) program:
* Sawtimber: wood biomass in the sawlog portion of the bole of timber species trees with diameter at breast height (DBH) ≥ 9" for softwoods (or DBH ≥ 11" for hardwoods), from a 1' stump up to a minimum top diameter of 7" for softwoods (or 9" for hardwoods).
* Pulp: wood biomass in the bole of timber species trees with DBH ≥ 5" from a 1' stump up to a minimum top diameter of 4", excluding any biomass that qualifies as sawtimber.
* Submerchantable: woody biomass (including its bark) in the tops and branches of all trees; all aboveground woody biomass of trees with DBH < 5"; and all aboveground woody biomass of woodland (i.e., non-timber) species.
* Nonmerchantable: biomass in foliage; bark on the bole; and stumps (wood and bark) of timber species trees with DBH ≥ 5".

Rather than predicting each product directly, the network predicts four ratios, and the products are calculated from those ratios and the input AGB:
1. The fraction of AGB in the merchantable bole (sawtimber plus pulp). The remainder of AGB is the residual (submerchantable plus nonmerchantable).
2. The fraction of the bole that is sawtimber. The rest of the bole is pulp.
3. The fraction of the residual that is nonmerchantable. The rest of the residual is submerchantable.
4. Boardfoot volume per metric ton of sawtimber biomass.

The three fractions are bounded between 0 and 1 and the boardfoot ratio is bounded to be greater than or equal to zero. As a result, the four biomass products are never negative and always sum exactly to the input AGB.

During training, each sample's error on each ratio is weighted by the size of the biomass pool the ratio divides (e.g., the error on the sawtimber fraction is weighted by bole biomass), so samples with more biomass in a pool contribute more to the fit for that pool. 

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

Biogeographic factors that influence tree form and the allocation of AGB into different product categories are addressed in the model by including forest type group and ecoregion as model inputs.

## Metrics
We evaluated model performance at subplot scale using National Forest Inventory data. Model performance is reported using Mean Absolute Error (MAE), Root Mean Squared Error (RMSE), mean bias, R-squared (R²), and Pearson's R. We also report the slope of a least-squares fit of observed on predicted values, which is 1 when predictions are neither compressed toward the mean nor spread too wide, and the ratio of the standard deviation of predictions to that of observations. Qualitative assessments were conducted through visual inspections of 2-D histograms of predicted versus observed values and comparisons of the distributions of predicted and observed values. Performance was also summarized separately for each forest type group and ecoregion.

## Evaluation data
### Datasets
We utilized inventory data from publicly-available FIA databases for both model training and evaluation. These inventory data were coupled with a global delineation of ecoregions published by [Dinerstein, et al. (2017)](https://doi.org/10.1093/biosci/bix014).

### Motivation
The FIA program provides a systematic and spatially balanced inventory of forests across CONUS. The geographic expanse and diversity, sample size, and revisit frequency provide a robust data source for model development and validation.

### Preprocessing
We summarized live tree attributes from the FIA annual inventory across CONUS for each forested subplot, including aboveground biomass, basal area, maximum tree height, and the biomass in each forest product category, along with canopy cover and forest type group from the subplot's conditions. Trees recorded outside the subplot boundary (e.g., large trees measured on macroplots) were excluded, as were subplots without live trees. The ecoregion for each subplot was defined using a spatial join of the fuzzed FIA plot coordinates with the global ecoregions layer.

## Training data
FIA subplots were split into training (70%), validation (15%), and test (15%) partitions. Partitions were assigned by 15 km grid tile, so that all subplots associated with an FIA plot, all revisits of a plot, and other plots that are nearby fall in the same partition. This limits the extent to which spatial autocorrelation between nearby samples inflates performance estimates. The validation partition was used for early stopping and checkpoint selection. The test partition was not used for model development, and was scored once for the release of this version.

## Data used at inference time
At inference time, we rely upon the VibrantForests ForestStructure model to produce wall-to-wall estimates of AGB, basal area, canopy cover, and maximum canopy height. These forest structure estimates were coupled with the ecoregions data layer and wall-to-wall estimates of Forest Type Groups from [FIA Bigmap](https://www.arcgis.com/home/item.html?id=4f78c504917a4f35b1c3191e94b5d565).

## Quantitative analyses
Performance on the held-out test partition (216,712 subplots) showed minimal bias for all products and high correlation between observations and predictions (Pearson R from 0.75 to 0.98). Most of the variance among samples was explained for the sawtimber, submerchantable, nonmerchantable, and boardfoot volume targets (R² from 0.88 to 0.96). A lower level of explained variance was observed for pulp (R²=0.57), and pulp predictions appear compressed toward the mean with variance among predicted values about three quarters of the spread among observed values. We believe this is related to the calculation of pulp biomass as the remainder of merchantable bolewood after subtracting sawtimber biomass.

| Target                    |   MAE |   RMSE |   Mean Bias |   R² |   Pearson R | Obs (Mean ± SD)   | Pred (Mean ± SD)   |
|:--------------------------|------:|-------:|------------:|-----:|------------:|:------------------|:-------------------|
| Sawlog Biomass (Mg/ha)    |  10.6 |   19.7 |       +0.05 | 0.94 |        0.97 | 43.5 ± 83.4       | 43.6 ± 82.2        |
| Pulp Biomass (Mg/ha)      |  10.5 |   18.3 |       -0.07 | 0.57 |        0.75 | 26.8 ± 27.8       | 26.7 ± 21.1        |
| Submerch Biomass (Mg/ha)  |   7.1 |   12.1 |       -0.03 | 0.88 |        0.94 | 38.7 ± 34.6       | 38.6 ± 32.5        |
| Nonmerch Biomass (Mg/ha)  |   2.3 |    4.4 |       +0.05 | 0.96 |        0.98 | 20.1 ± 21.8       | 20.2 ± 21.0        |
| Boardfoot Volume (MBF/ha) |   4.9 |   10.2 |       -0.07 | 0.94 |        0.97 | 18.5 ± 42.9       | 18.5 ± 43.4        |

| ![Observed against predicted forest products on the test partition](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantForests/ForestProducts/2.0.0/observed_against_predicted_test.png) |
| :-- |
| The top row displays 2-D histograms of observed against predicted values for each product on the test partition, with higher densities of samples in lighter colors. The 1:1 line indicating perfect correspondence between predictions and observations is shown as a black dashed line, and a least-squares fit of observed on predicted values as a red line. The bottom row compares the distributions of observed and predicted values. |

## Ethical considerations
The data used and generated by ForestProducts are not considered sensitive nor to pose substantial risks to human health or safety. The data are intended to be instrumental in decision-making that may indirectly enable human health and safety to be better protected through more cost-effective and targeted wildfire and forest restoration planning. Similar data sources already exist at lower resolution and periodic update frequency produced by the US Forest Service (e.g., [TreeMap by Riley et al. 2021](https://www.nature.com/articles/s41597-020-00782-x)), and more precise maps of forest product volumes are commonly generated in public and private sectors based on lidar data.

We are conscious of the fact that forest protection and conservation communities in the USA and abroad are concerned about exploitation of forests in a manner that is myopically focused on timber utilization at the expense of environmental and social costs. Given the preexisting availability of similar forest product data within the timber industry, by state natural resource agencies, and at the US Forest Service, we do not believe the provision of forest product data from publicly-available satellite imagery on a regular cadence is going to meaningfully alter the behavior of public or private land managers in terms of increasing or decreasing harvest levels. We are hopeful that more consistent data on forest product volumes allows the judicious management of wood resources to be incorporated alongside the variety of other forest and community outcomes that can be evaluated and acted upon through the decision support systems we develop. 

## Caveats and recommendations
This model was developed and evaluated with an initial focus on the contiguous USA. Caution is advised against naive application beyond that scope without additional effort to update the training and evaluation datasets to determine model performance in other regions.

Although there remains significant interest (and debate) surrounding salvage logging of dead or downed trees following natural disturbances, this model has not been trained to estimate the merchantability of dead trees, and is unlikely to provide accurate estimates of merchantable volume in the wake of acute mortality events like wildfire, extreme wind, or severe pest outbreaks. 

## Acknowledgements
The development of ForestProducts was supported by funding from a grant by the Doris Duke Foundation to American Forests for the development of the Forest Innovation Platform.

This model card format was adapted from a Markdown template developed by [Christian Garbin](https://github.com/fau-masters-collected-works-cgarbin/model-card-template), based on sections and prompts from [Mitchell et al. (2019)](https://arxiv.org/abs/1810.03993).
