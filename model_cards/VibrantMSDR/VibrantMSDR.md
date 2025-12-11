# VibrantMSDR: Model Card
Jump to section:
- [Overview](#overview)
- [Model details](#model-details)
- [Intended use](#intended-use)
- [Factors](#factors)
- [Metrics](#metrics)
- [Data](#data)
- [Training process](#training-process)
- [Inference process](#inference-process)
- [Quantitative analyses](#quantitative-analyses)
- [Ethical considerations](#ethical-considerations)
- [Caveats and recommendations](#caveats-and-recommendations)
- [Acknowledgements](#acknowledgements)

## Overview
VibrantMSDR is a regression model developed by [Vibrant Planet](https://www.vibrantplanet.net/) to estimate the Maximum Size-Density Relationship (MSDR) of trees in forests based a forest's composition and location from which the maximum Stand Density Index (SDI<sub>max</sub>) and Relative Density (RD) may be derived. The MSDR represents the expected maximum density of stems per area as average tree size increases. SDI<sub>max</sub> is the point on the MSDR line using a reference tree diameter of 10 in. (25.4 cm). A forest's density in relation to the MSDR or SDI<sub>max</sub> is commonly used to infer the likelihood of observing density-driven mortality and to indicate forest vulnerability to changes such as drought, insects, and/or disease.

VibrantMSDR combines two modeling stages. The first learns to estimate SDI<sub>max</sub> from forest inventory data and generates a lookup table. The second applies SDI<sub>max</sub> values from the lookup table over raster inputs to generate wall-to-wall outputs.

### Contributors
David Diaz, Vincent Landau, Luke Zachmann, Tony Chang, Nathan Rutenbeck

### Model date
November 2025

### Model version
1.0.0

### Model type
Bayesian Hierarchical Linear Mixed Quantile Regression Model

## Model details
VibrantMSDR fits a linear equation to estimate MSDR lines by predicting stem density observed across forest plots as a function of average tree size, forest composition, and region:  

$$  
\ln N_i = \sum_{c \in C} \left[\alpha_{r,c} + \beta_{r,c} * \ln D_i \right] * \omega_{c,i}
$$  

where: 

* **$N_i$** is the trees per hectare observed across plot $i$
* **$c$** is each distinct component (e.g., species, forest type) observed on a single plot, out of all components **$C$** observed on that plot 
* **$\alpha_{r,c}$** is the intercept term for region $r$ and component $c$
* **$\beta_{r,c}$** is the slope term for region $r$ and component $c$
* **$D_i$** is the quadratic mean diameter of live trees across plot $i$
* **$\omega_{c,i}$** is the proportion of stocking on plot $i$ attributed to component $c$

The model was defined to calculate $\alpha_{r,c}$ and $\beta_{r,c}$ as the sum of global baseline slope and intercept terms plus random effects for each component $c$, each region $r$, and allowing for localized component effects across regions $rc$. The global baseline slopes and intercepts represent default values across all regions and components. Random effects are then added for each component $c$ (i.e., forest type group) and each region $r$, and components-among-regions $rc$. That is:  

$$  
\alpha_{r,c} = \alpha + \epsilon_r + \epsilon_c + \epsilon_{rc}  
$$  

$$  
\beta_{r,c} = \beta + \gamma_r + \gamma_c + \gamma_{rc}
$$  

where:
* **$\alpha, \beta$** are global baseline intercept and slope parameters, respectively
* **$\epsilon_r , \epsilon_c, \epsilon_{rc}$** are random effects on the intercept of the MSDR line for each region $r$, each component $c$, and components-across-regions $rc$. 
* **$\gamma_r , \gamma_c, \gamma_{rc}$** are random effects on the slope of the MSDR line for each region $r$, each component $c$, and components-across-regions $rc$. 

This approach of using incremental random effects permits the derivation of a regional-baseline for SDI<sub>max</sub> value that may be used when the composition of the forest is unknown, permitting wall-to-wall inference across all regions without requiring forest composition data.

We fit the MSDR line as a Bayesian Quantile Regression with the likelihood defined with an Asymmetric Laplace Distribution (ALD) using quantile parameterization:  

$$  
MSDR \sim ALD(\mu, \sigma, q)
$$  

where:
* **$\mu$** is the natural log of estimated trees per hectare on plot $i$ (i.e., $\ln\hat{N_i}$)
* **$\sigma$** is the scale term of the ALD distribution (a parameter to be learned)
* **$q$** is the quantile to be targeted, defined as $q=0.9$

The random effects on slope and intercept terms for components, regions, and components-across-regions were all defined as ZeroSumNormal distributions. This approach is similar to using partial-pooling of hierarchical parameters for regions and components and components-across-regions, but allows the incremental effects associated with each region and component to be explicltly seperated. 

## Intended use

### Primary intended uses
VibrantMSDR was primarily developed to provide a springboard from which RD could be estimated, describing the current stocking of forests in relation to their maximum potential stocking at any point in time (i.e., SDI<sub>t</sub> / SDI<sub>max</sub>). Estimates of SDI at any point in time are intended to provided by remote sensing models of forest structure.

| ![CONUS Relative Density Map](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantMSDR/1.0.0/rsdi_conus_map.png) |
| :-- |
| *Example of intended use to map Relative Density based on combination of VibrantForests estimate of SDI from 2024 imagery and VibrantMSDR estimate of SDI<sub>max</sub>. Labels in the map legend correspond to: LDM = Low Density Management; ZODM = Zone of Optimal Density Management; ZICM = Zone of Imminent Competition Mortality; and DDM = Density-Dependent Mortality. These labels were adapted from <a href="https://research.fs.usda.gov/treesearch/67075"> Ray et al. (2023)</a> and <a href="https://research.fs.usda.gov/treesearch/68854">Zukswert and Kenefic (2025)</a>* |


The outputs of VibrantMSDR are intended to support forest mapping and monitoring with direct applications in the [Vibrant Planet Platform](https://www.vibrantplanet.net/platform), a decision support system for wildfire risk assessment and land management planning. The physiological limits represented by MSDR and SDI<sub>max</sub> are generally not expected to shift dramatically year-to-year, but rather to reflect intrinsic qualities of tree physiology alongside climatic and edaphic constraints. As such, VibrantMSDR is intended to be updated periodically for enhancements in modeling techniques and over time as the carrying capacity of forests evolves and adapts with climate change. 

### Primary intended users
The primary intended users of VibrantMSDR predictions are natural resource managers and professionals leading the development of wildfire risk assessments, community wildfire protection planning, and forest restoration plans through the [Vibrant Planet Platform](https://www.vibrantplanet.net/platform). These use cases generally involve access to zonal summaries of these rasters at the scale of management units rather than accessing the underlying 10m resolution rasters directly.

VibrantMSDR is being used to produce RD rasters that will be served through the [Forest Innovation Platform](https://forestinnovationplatform.org) under development by American Forests to support rapid assessments of forest vulnerability and adaptation planning. We also expect to release rasters of RD covering the contiguous United States (CONUS) at 30m resolution through [Vibrant Planet Data Commons](https://www.vpdatacommons.org/). We anticipate these raw rasters may be used by forest analysts and researchers for a variety of forest health and vulnerability analyses.

### Out-of-scope use cases
Although we anticipate uses of VibrantMSDR raster outputs by forest researchers and analysts in the public and private sectors for regional- or national-scale analyses, we have not substantially altered model development based on these potential use cases.

Similarly, this model was not intended for use in individual tree detection or attribution. The 10m resolution is intentionally selected for representing forest attributes on an areal basis as opposed to individual tree detection and attribution.

## Factors
The key factors involved in the development of VibrantMSDR are the definitions of regional boundaries (e.g., ecoregions), the availability of forest inventory data, and the definition of components categories (i.e., tree species, forest types, forest type groups) for which MSDR coefficient are estimated. 

## Metrics
We evaluate the performance of VibrantMSDR at regional scales using distributions of observations from National Forest Inventory data that were held out from model training. We compare our estimates of SDI<sub>max</sub> with empirical observations of SDI (targeting the 90<sup>th</sup> percentile of these observations). We also compare our estimates of SDI<sub>max</sub> to the All-Live-Tree Stocking Percent (ALSTK) attribute generated by the FIA program. 

ALSTK is estimated using a series of equations that estimate per-tree stocking values based on the tree's species, size, and expansion factor, recommended stocking levels, adjustments based on competitive position among trees, and other aspects, as defined in [Arner et al. (2001)](https://www.fs.usda.gov/fmsc/ftp/fvs/docs/gtr/Arner2001.pdf). ALSTK is truncated at 120%. ALSTK is summarized into the following categories: 

| ALSTK Range | Description |
| :-- | :-- |
| >100% | Overstocked |
| 60-100% | Fully Stocked |
| 35-60% | Medium Stocked |
| 10-35% | Poorly Stocked |
| <10% | Nonstocked |

These zones are shifted somewhat relative to the zones suggested for interpreting RD introduced by [Drew and Flewelling (1979)](https://doi.org/10.1093/forestscience/25.3.518) and captured in a recent review by [Chivhenge et al. (2024)](https://link.springer.com/10.1007/s40725-024-00212-w), where crown closure occurs at RD of 15%, the recommended lower limit of density management at RD of 30%, the onset of imminent competition mortality at RD 55% and maximum size-density at RD of 100%. These zones also differ slightly from those employed in the graphic of CONUS RD above.

We visualize and inspect model performance for each component in each region by plotting the average tree diameter versus observed stem density for plots that are single-component (e.g., single forest type group across the plot) overlaid by the MSDR line. 

## Data

VibrantMSDR relies upon four primary datasets: ecoregion boundaries; forest inventory data (during training only); and maps of forest density and composition (during inference only). 

Regional boundaries are used for both training and inference. We employed the Resolve Ecoregions 2017 delineation of global ecoregions by [Dinerstein, et al. (2017)](https://doi.org/10.1093/biosci/bix014).

During training, the model relies upon inventory measurements from the US National Forest Inventory, known as the Forest Inventory & Analysis (FIA) program. We utilize observations of tree density and size for live trees on plots established by the FIA program across CONUS. To define the *components* of each plot, we utilize Forest Type Groups defined in the FIA program. At inference time, the model relies upon the [FIA Bigmap Forest Type Group](https://www.arcgis.com/home/item.html?id=4f78c504917a4f35b1c3191e94b5d565) layer to define the expected composition for each pixel.

### Preprocessing
We summarized FIA observations of stem density and tree size to the plot scale (each FIA plot is a cluster of four subplots) under the expectation that the carrying capacity of a forest condition represented by the MSDR line is an emergent property at that is expressed at scales larger than individual subplots. To reflect the fact that SDI<sub>max</sub> is generally considered a physiological attribute of individual tree species, or forest types, we also calculated the proportion of plot-level basal area for each Forest Type Group observed across the FIA plot. Forest Type Groups are defined within each of one or more "conditions" delineated across a cluster of FIA subplots. The vast majority of FIA plots have only a single Forest Type Group observed across the four subplots.  

| ![FIA Plot Layout](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantMSDR/1.0.0/fia_plot_layout.png) |
| :-- |
| *Illustration of FIA plot layout with four sublots and three conditions. SDI and SDI<sub>max</sub> are calculated (summed) and predicted at the spatial scale of a whole plot.* |

We performed a spatial join of each plot to the ecoregions data layer using the fuzzed coordinates of each FIA plot center assign each plot to a single region.


We employ the [FIA Bigmap Forest Type Group](https://www.arcgis.com/home/item.html?id=4f78c504917a4f35b1c3191e94b5d565) dataset to generate wall-to-wall estimates of forest composition across CONUS. We resampled the FIA Bigmap dataset to 10m resolution, reprojected to EASE Grid North, and retiled the rasters into 15km squares. Several pixel values not corresponding to defined Forest Type Groups were present in the original data layer. These were generally remapped to the nearest Forest Type Group value, or in some cases switched to "Nonstocked". Within each tile, we replaced pixels classified as "Nonstocked" with the value of the nearest neighbor that had a Forest Type Group. If no neighbors were found, the pixels remained as classified as "Nonstocked." We then replaced all NoData pixels in the Bigmap layer with the "Nonstocked" category. This enables wall-to-wall prediction, and where Nonstocked pixels are encountered, the baseline + regional SDI<sub>max</sub> value is used without further adjustment taking into account Forest Type Group. In cases where Forest Type Group has values other than Nonstocked, additional positive or negative effects for that forest type are incorporated into the SDI<sub>max</sub> estimation.

### Features
During training, the features used as inputs to VibrantMSDR are the quadratic mean diameter of live trees on each plot, the proportion of live tree basal area for each plot component, and the region of the plot. During training, the inputs are tabular.

At inference time, the features are provided as raster arrays. We utilize estimates of current SDI and Forest Type Group from two separate models.  We derive SDI from the predictions of basal area and quadratic mean diameter produced by VibrantForests, and predictions of forest composition adapted from the FIA Bigmap Forest Type Group layer. 

### Targets
At training time, the target for VibrantMSDR is the 90<sup>th</sup> percentile of the natural log of trees per hectare (TPH). This metric is also translated into an estimate of SDI<sub>max</sub> as $e^{\alpha + \beta*25.4}$ where 25.4 cm is the standard reference diameter used in calculating SDI. The training process generates a lookup table with SDI<sub>max</sub> values for each component, region, and baseline regional SDI<sub>max</sub> values for cases where the composition of a forest is not known.

At inference time, the ONNX implementation of this model generates four output arrays: SDI, SDI<sub>max</sub>, RD, and TPH. 

## Training process
The MSDR model was defined in the [Numpyro]( 	
https://doi.org/10.48550/arXiv.1912.11554) probabilistic programming library. It was fitted using Stochastic Variational Inference to identify the *maximum a posteriori* values of the slope and intercept terms for each component and region. During initial model development, we noticed that the combination of very large sample size and sensitivity of the AsymmetricLaplaceQuantile distribution with quantile targets near 1.0 led to slow and difficult sampling using Markov Chain Monte Carlo.

We divided the data into train (80%) and validation (20%) sets to run Variational Inference. We subdivided the plots into partitions using stratified sampling to ensure equal representation of regions and composition in each partition, and assigning all observations of each plot to either the train or validation partition (i.e., when a single plot been visited more than once every 5-10 years, all observations at that location were assigned to a single partition). We trained the model for 200,000 steps using the Adam optimizer and monitored the negative log likelihood on the train and validation partitions each step. We ran the model until convergence, and selected the coefficients from the step in training that achieved the lowest loss on the validation set.

The SDI<sub>max</sub> values for each forest type group in each ecoregion and the baseline + regional SDI<sub>max</sub> effect were exported and used a lookup table in a separate model created to run inference across rasters.

## Inference process
Predictions are made using a model defined in PyTorch to generate wall-to-wall predictions with rasters as inputs and outputs. Inference is performed over 15km * 15km tiles at 10m resolution. 

SDI and trees per hectare are derived for each pixel directly from the estimates of basal area and quadratic mean diameter produced by VibrantForests. The model looks up the SDI<sub>max</sub> for each pixel at 10m resolution and calculates RD as SDI / SDI<sub>max</sub>. 

The model then passes a circular smoothing kernel with radius of 40m over the SDI<sub>max</sub> and RD layers to calculate averages of these attributes centered at each pixel. RD is clamped to the range of 0-200. This smoothing reflects our expectation that carrying capacity and RD emerge at scales comparable to the extent of an FIA plot and the scale at which these stocking limits were learned. SDI and TPH layers are not smoothed.

## Quantitative analyses
We visualized MSDR lines for every Forest Type Group in every ecoregion to confirm the model behaved as expected. The plots display observed tree size and density for each plot 100% covered by that Forest Type Group. This allows us to validate the MSDR lines against "pure" samples of each Forest Type Group in each region. An example for the California Mixed Conifer Forest Type Group is shown here. The MSDR lines for all forest types and regions showed similar characteristics. 

| ![Maximum Size-Density Relationships for California Mixed Conifer](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantMSDR/1.0.0/msdr-california_mixed_conifer.png) |
| :-- |
| *Example fit of MSDR lines for plots covered only by California Mixed Conifer Forest Type Group, across all the ecoregions where this Forest Type Group is observed. MSDR lines in regions where few samples are observed are strongly influenced by the hiearchical regularization induced by the Bayesian model which pulls the slopes and intercepts towards the regional and forest type averages* |

### Regional effects
VibrantMSDR generated regional effects that appear primarily related to the influence of aridity on forest density. These effects can be seen both in isolation of the regional random effects on the slope and intercept of the MSDR line, and when viewed geographically.
| ![Regional Maximum Stand Density Index](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantMSDR/1.0.0/regional_sdimax.png) |
| :-- |
| *Illustration combining the baseline SDI<sub>max</sub> with regional effects.* |

The coastal temperate rainforest ecoregions across California, Oregon, and Washington stand out as the regions with highest SDI<sub>max</sub> while the deserts across the southwestern borderlands stand out as the regions with the lowest SDI<sub>max</sub>.  

| ![Regional effects on SDImax](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantMSDR/1.0.0/regional_effects.png) |
| :-- |
| *The combined regional random effects on MSDR slopes and intercepts translated into proportional change in baseline SDI<sub>max</sub>. Effects for regions that increase SDI<sub>max</sub> by 10% or more are shown in blue and those that decrease SDI<sub>max</sub> by more than 10% or shown in red. Regions with effects on SDI<sub>max</sub> smaller than 10% are shown in gray.* |

### Composition effects
VibrantMSDR learned random effects on SDI<sub>max</sub> associated with Forest Type Groups where types that commonly generate very dense conditions (often referred to as "doghair") showed positive random effects while those that occur in more sparse woodland and savannah settings showed negative random effects.  

| ![Composition effects on SDImax](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantMSDR/1.0.0/component_effects.png) |
| :-- |
| *The combined component random effects on MSDR slopes and intercepts translated into proportional change in baseline SDI<sub>max</sub>. Effects for Forest Type Groups that increase SDI<sub>max</sub> by 10% or more are shown in blue and those that decrease SDI<sub>max</sub> by more than 10% or shown in red. Forest Type Groups with effects on SDI<sub>max</sub> smaller than 10% are shown in gray.* |

### Evaluation against field observations
To evaluate the estimates of SDI<sub>max</sub> and RD generated by VibrantMSDR, we consider two attributes from the FIA dataset: All-Live-Tree Stocking Percent (ALSTK); and the empirical 90<sup>th</sup> percentile of observed SDI values among compositionally-pure plots in each ecoregion. The details behind these methods are described further below.

| ![Modeled estimates versus FIA observations](https://vp-open-science.s3.us-west-2.amazonaws.com/model_cards/assets/VibrantMSDR/1.0.0/estimates_vs_fia.png) |
| :-- |
| *The heatmap on the left shows the count of FIA plots within hexagonal bins spanning 1 percentage point of ALSTK and RD. The scatter plot on the right shows the correspondence of observed and estimated SDI quantiles with points representing each Forest Type Group in each global ecoregion with the size of the scatter point scaled by the number of samples of that type in that region.* |

#### All-Live-Tree Stocking Percent
First, we compare the estimated RD for each plot and compare it against ALSTK estimates from the FIA dataset.

The comparison of our estimated SDI<sub>max</sub> with FIA's estimate of ALSTK is made at the scale of FIA plots, and allows for all plots to be compared including those with more than one Forest Type Group observed. We observed good correspondence between our estimate of RD and FIA's estimate of ALSTK.

#### Empirical SDI Percentile
To evaluate our estimates of SDI<sub>max</sub> with observed values of SDI, we aggregated plot-level SDI for Forest Type Group and global ecoregion, and calculated the 90<sup>th</sup> percentile of observed SDI. This is similar to the target we employed in our quantile regression (although VibrantMSDR was not directly trained to predict the 90<sup>th</sup> percentile of SDI, but rather the entire MSDR line that covered 90% of observed stem densities). This aggregation excludes plots with more than one Forest Type Group. 

When comparing estimated vs. observed SDI<sub>max</sub> values in this way, we notice that Forest Type Groups that have few observations in an ecoregion (the small points in the scatter plot) tend to drift above the 1:1 line. This reflects the hierarchical regularization of the Bayesian model, pulling the MSDR line coefficients for these rarer combinations of forest types and regions towards the average coefficients for each region and Forest Type Group. We observed a tight correspondence between the empirical 90<sup>th</sup> percentile of SDI for each forest type and ecoregion that had substantial sample sizes.

## Ethical considerations
The data used and generated by VibrantMSDR are not considered sensitive nor to pose substantial risks to human health or safety. The data are intended to be instrumental in decision-making that may indirectly enable human health and safety to be better protected through more cost-effective and targeted wildfire and forest restoration planning. Similar data sources already exist at lower resolution produced by academic research groups (e.g., [Chivhenge et al. (2025)](https://www.nature.com/articles/s41597-025-06012-6), [Kimsey et al. (2019)](https://www.sciencedirect.com/science/article/pii/S0378112718317122)), and we do not anticipate the release of higher resolution data to pose any substantial additional risks.

## Caveats and recommendations
VibrantMSDR has been developed and evaluated with an initial focus on the contiguous USA. We caution against naive application of this model beyond that scope without additional effort to update the training and evaluation datasets to determine model performance in other regions.

### Limitations to precise interpretation and thresholds
Although MSDR lines are intended to reflect a theoretical maximum, in practice individual observations and predictions above these values are regularly encountered. This should be expected considering the MSDR lines we fit targeted the 90<sup>th</sup> percentile of field observations (i.e., at least 10% of observations should be expected to have higher density). We caution against interpretation of small changes in RD, as the precision of these estimates is not sufficient to detect thresholds at the scale of individual percentages. We encourage use of RD using ordinal levels corresponding to density and management "zones". We provide the underlying RD percentage value to allow users to define their own levels and zones given the wide variation in how those zones are defined. Given the closeness of VibrantMSDR's correspondence with the values of FIA ALSTK, using those zones to interpret VibrantMSDR's estimates of RD would be a good option. 

### Potential for underestimation of mortality risks among large trees
MSDR lines and SDI<sub>max</sub> values were originally developed under the premise that density-related mortality occurs consistently as forests approach the MSDR line regardless of the average tree size. However, many forest type groups show maximum stem densities that pull away from the MSDR line as average tree sizes grow larger. This has been referred to as a "Mature Stand Boundary" [(Shaw and Long, 2007)](https://research.fs.usda.gov/treesearch/26787) which indicates that stands with larger trees may not be as efficient at re-occupying growing space following tree mortality compared to stands with smaller trees. This may indicate that stands with larger trees will begin experiencing mortality at lower levels of Relative Density than stands of smaller trees. The predictions of SDI<sub>max</sub> should thus be expected to generate conservative estimates of Relative Density for large stands that may be biased lower than the actual vulnerability of these stands to drought, insects, or disease.

### Artifacts at ecoregion boundaries
In its current form, the model may generate abrupt steps in estimates of SDI<sub>max</sub> across ecoregion boundaries that are more pronounced in cases where a Forest Type Group is not defined. This is more commonly observed in woodland areas that FIA Bigmap often represents as NoData. This effect is more muted, but not absent, in Relative Density predictions. These ecoregion effects will be most pronounced crossing ecoregion boundaries where major transitions occur in aridity (e.g., the transitions across the Cascade and Sierra Nevada ranges). These artifacts mean that planning activities crossing multiple ecoregion boundaries should be preceded by inspection to determine how much precision in Relative Density values can be relied upon to differentiate between treatment areas for prioritization. In future updates to the model, we intend to shift to higher-resolution representations of regional effects (e.g., by using down-scaled measures of aridity or soil moisture availability).

### Artifacts around the built environment
At inference time, VibrantMSDR is applied using a smoothing kernel across estimates of SDI and SDI<sub>max</sub>. In the current workflow, we generate estimates of SDI from VibrantForests, which may produce artifically high estimates of SDI over the built environment. We currently apply a land cover layer to zero-out predictions of Relative Density over pixels with built-up, water, and agricultural vegetation types, but this zeroing occurs after the smoothing kernel has already been applied. In some dense urban areas with skyscrapers, we have observed rings around buildings that are an artifact we expect to address in subsequent enhacenments to VibrantForests and to our land cover data sources.

## Acknowledgements
The development of VibrantMSDR was supported by funding from a grant by the Doris Duke Foundation to American Forests for the development of the Forest Innovation Platform.

This model card format was adapted from a Markdown template developed by [Christian Garbin](https://github.com/fau-masters-collected-works-cgarbin/model-card-template), based on sections and prompts from [Mitchell et al. (2019)](https://arxiv.org/abs/1810.03993).