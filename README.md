## Code to reproduce findings from [A tale of ten cities: data bias in human mobility is pervasive and highly location specific](https://link.springer.com/article/10.1140/epjds/s13688-026-00685-8)

___

# Data 
**We use GPS mobility data collected from smartphones by one mobility data provider who requested to be anonymous.** They deliver anonymized records of GPS locations from users who have opted-in to provide access to their location data, while remaining compliant under the General Data Protection Regulation and the California Consumer Privacy Act. Data is collected as GPS pings, which has been privacy enhanced before being shared with us. This includes the removal of mobility points which fall close to sensitive locations such as healthcare sites, child related facilities, churches and other religious facilities, federal and government buildings, military facilities, payday loan services, and facilities related to ethnic minorities. In addition, the data provider further protects user privacy by algorithmically inferring home and work locations, and rather than sharing the precise location of these, all data points in the vicinity of these areas are transformed (uplevelled) to the centroid of the Census Block Group they are located in.

Below we outline the code provded in various notebooks which were used to proces, clean, aggregate, and model the data.

___

# `1_extract_US-level.ipynb` 

**This notebook extract datapoints and home locations of users in US during the period of interest.** This notebook was run on the data provider's inhouse compute cluster and processes and saves data inhouse on their servers. 

**WHEN: Use this notebook once per _period of interest (poi)_.**  
Outputs can be used for all regions of interest within the US in the chosen poi.

### Inputs
User data tables (from non-disclosed data provider): 
- table: **device_location_upleveled**
- table: **device_recurring_area**

### Outputs
- `datapoints/US_`***yyyy_mm***`.csv`, US level; all measured datapoints, aggregated per user in the US
- `home_locations/US_`***yyyy_mm***`.csv`, US level; inferred home locations of users 



_____

# `2_couple_users_homelocs.ipynb`

**This notebook create mappings between users, their datapoint contribution and inferred home census tract.** This notebook was run on the data provider's compute cluster. No data left their inhouse compute environment. 

**WHEN: Use this notebook once per _region of interest (roi)_ and _period of interest (poi)_.**  

### Inputs
- `home-locations/US_yyyy_mm.csv`, from `extract_prep_US_level.ipynb`
- `datapoints/US_yyyy_mm.csv`, from `extract_prep_US_level.ipynb`

### Outputs
- `dpoints-homelocs/roi_yyyy-mm.csv`, roi level; all measured users with home location in NYC, with home location census tract and number of contributed datapoints

___

# `3_model_dataprep.ipynb`

**This notebook matches users and their data contribution to demographics of their home location, leading to median datapoint contributions and demographics per census tract.** 
The code was run on the data provider's inhouse compute cluster. 

**WHEN: Use this notebook once per _region of interest (roi)_ and _period of interest (poi)_.**  

### Inputs

* `dpoints_homelocs/roi_yyyy_mm.csv`, from notebook `couple_users_homelocs.ipynb`
- Census gov ACS: <a href="https://data.census.gov/table?q=S1701+2019+ACS&g=050XX00US36005,36061,36081,36085,36047">S1701</a> | __Poverty table__  
  \> `Main/census-data/poverty-data/`*roi*`/ACSST5Y`*yyyy*`.S1701-Data_CensusTracts`*roi*`.csv` 
* Census gov ACS: <a href='https://data.census.gov/table?q=2021+S0101&g=0100000US$1400000'>S0101</a> | __Age and Sex table__   
  \> `Main/census-data/age-sex-data/ACSST5Y`*yyyy*`.S0101-Data_CensusTracts`*roi*`.csv` 
- Census gov ACS: <a href='https://data.census.gov/table?q=2021+S1501&g=0100000US$1400000'>S1501</a> | __Educational Attainment and Ethnicity table__  
  \> `Main/census-data/education-data/ACSST5Y`*yyyy*`.S1501-Data_CensusTracts`*roi*`.csv` 


### Outputs

- `stats/demographics/roi_yyyy_mm.csv`, roi level; overview of census tracts' total and median datapoints + demographics (poverty, age, sex, educational attainment, ethnicity).



____

# `4a.model_RF.ipynb` and `4b_model_RF_leave1out.ipynb`

**This notebook builds a random Forest model with nested cross-validation for tuning hyperparameters.**
The notebook was inhouse on the data provider's compute cluster.


### Inputs

- `stats/demographics/roi_yyyy_mm.csv`, from notebook `model_dataprep.ipynb`


### Outputs

- `stats/model_outputs/R2s.txt`, addition of one line to this overview file of all rois; roi + poi and R2
- `stats/model_outputs/roi_yyyy_mm`, pickled model list (10 models)
- `stats/model_outputs/y_preds/roi_yyyy_mm`, y_preds is a tuple per census tract of predicted and observed y per census tract
- `stats/model_outputs/SHAP_values/roi_yyyy_mm.csv`, SHAP values
- `stats/model_outputs/X_feature_vals/roi_yyyy_mm.csv`, feature values in same order as SHAP values (for SHAP plot)


## Model runs pipeline (Random Forest)

_Overview of the necessary steps to create models on datapoint contributions per census tract of region of interest (roi): datasets to be downloaded and list of notebooks with their inputs/outputs._

`1_extract_US-level.ipynb` > `2_couple_users_homelocs.ipynb` > `3_model_dataprep.ipynb` > `4a_model_RF.ipynb` + `4b_model_RF_leave1out.ipynb` > `5_plots_model.ipynb`


## Model preparations
#### Exports from Census gov

- Census gov ACS: <a href="https://data.census.gov/table?q=S1701+2019+ACS&g=050XX00US36005,36061,36081,36085,36047">S1701</a> | __Poverty table__  
  \> `Main/census-data/poverty-data/`***roi***`/ACSST5Y`***yyyy***`.S1701-Data_CensusTracts`*roi*`.csv` 

* Census gov ACS: <a href='https://data.census.gov/table?q=2021+S0101&g=0100000US$1400000'>S0101</a> | __Age and Sex table__   
  \> `Main/census-data/age-sex-data/ACSST5Y`***yyyy***`.S0101-Data_CensusTracts`***roi***`.csv` 

- Census gov ACS: <a href='https://data.census.gov/table?q=2021+S1501&g=0100000US$1400000'>S1501</a> | __Educational Attainment and Ethnicity table__  
  \> `Main/census-data/education-data/ACSST5Y`***yyyy***`.S1501-Data_CensusTracts`***roi***`.csv` 

*In census gov table:  
poi (period of interest): Select year.  
roi (region of interest): Select GEO's: all census tracts of the roi's counties.*

- `tab20_blkgrp20_blkgrp10_natl.txt`  
Census gov relationship file for converting between 2010 and 2020 block groups.    
Relationship file for NY state, can be found <a href=https://www2.census.gov/geo/docs/maps-data/data/rel2020/blkgrp/tab20_blkgrp20_blkgrp10_natl.txt>here</a>. General US database for conversion files can be found <a href=https://www2.census.gov/geo/docs/maps-data/data/rel2020/>here.</a>

___


# `5_extract_stops.ipynb`

**This notebook extracts stops locations for a region of interest and matches stops to block group level**
This code was run on the data provider's internal compute cluster. No data left the environment.


### Inputs
- table: **stop_uplevelled** 
- `user-poverty-maps/roi_yyyy_mm.csv`, from notebook `create_pov_map.ipynb`

### Outputs
- `nw_extracts/Stops_yyyy_mm.csv`, per user stop: user_id, block_group_id, stop_zoned_datetime, dwell_time_minutes
   - N.B. Stops from *roi* citizens, but can be stops outside *roi* as well.
   - Now in notebook already selection for users with at least 30 stops.

___

# `6_extract_stops.ipynb`

**This notebook takes extracted stops and constructs a aggregated mobility network.**
We divide users into 20 data production groups (bottom 5%, 5-10%, 10-15%, ... top 5% data producers). A network is constructed for each group of data producers.

Nodes represent visited locations at the block group level. Their weight represents aggregated unique user visits. The directional edges represent the aggregated number of trips between locations, with their weight representing the total trips in that direction. We defined a ‘trip’ between census tracts as a transition between temporally ordered GPS positions, where a preceding point is located in census tract A, and the next is in census tract B. We they say the device has made a ‘trip’ between census tracts A and B. Mobility networks are constructed by summing all these transitions up.

This code was run on the data provider's internal compute cluster. No data left the environment.
We only focus on New York City here. 

### Inputs
- `nw_extracts/Stops_yyyy_csv`, per user stop: user_id, block_group_id, stop_zoned_datetime, dwell_time_minutes

### Outputs 
- `nw_extracts/NWs_NYC_yyyy_mm_edges.csv`
- `nw_extracts/NWs_NYC_yyyy_mm_nodes.csv`


___

# `7_extract_backbone.ipynb`

**This code extract the network backbone of networks constructed by different data production groups**


### Inputs
- `nw_extracts/NWs_NYC_yyyy_mm_edges.csv`
- `nw_extracts/NWs_NYC_yyyy_mm_nodes.csv`

### Outputs
- Raw networks, and noise corrected backbone networks for groups: bottom 10%, bottom 20%, top 20%, and top 10% data producing groups.


___

# `8_manuscript_figures.ipynb`

**This notebook plots the various figures from the paper**

