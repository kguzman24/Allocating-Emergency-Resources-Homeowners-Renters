# DS 4320 Project 2: Allocating Emergency Resources Homeowners vs. Renters in Flood/Hurricane Affected Communities

**Executive Summary:** This repository contains the full pipeline for analyzing Federal Emergency Management Agency (FEMA) housing assistance distribution equity for homeowners vs. renters in flood/hurricane affected communities. The project combines two datasets: the FEMA `housing-assistance-program-data-owners-v2` and `housing-assistance-program-data-renters-v2`. Both homeowner and renter data were found on the OpenFEMA Dataset: Housing Assistance Program Data webpage. Data was loaded into the `Pipeline.ipynb` notebook through each dataset's API endpoint. It was converted into pandas DataFrames for analysis and filtered to only include Hurricane/Flood disasters to scope the dataset to a single event for homeowner vs renter comparison (event 4827 - Hurricane Helene). Additionally, the license and press release are displayed in the root repository, alongside an exploratory data analysis jupyter notebook entitled `EDA.ipynb`.

**Name:** Karen Guzman

**NetID:** Sae3gg

**DOI:** https://doi.org/10.5281/zenodo.19870690

[**Press Release**](https://github.com/kguzman24/Allocating-Emergency-Resources-Homeowners-Renters/blob/main/PressRelease)

[**Pipeline**](https://github.com/kguzman24/Allocating-Emergency-Resources-Homeowners-Renters/blob/main/Pipeline.ipynb)

[**MIT LICENSE**](https://github.com/kguzman24/Allocating-Emergency-Resources-Homeowners-Renters/blob/main/LICENSE)

## Problem Definition

### Problem

General Problem: Allocating emergency response resources

Specific Problem: Is FEMA housing assistance equitable in how it distributes to low-income renters in flood/hurricane affected communities, compared to homeowners?

### Motivation for this Project
The aftermath of hurricanes and floods, the households that suffer the most, like low-income renters, people with disabilities, and communities of color, are frequently less serves by disaster recovery systems. FEMA's Individual Assistance housing programs are the main federal mechanism that help families rebuild. However, approval rates and aid amounts vary drastically across income levels. Understanding and redesigning how housing resources are allocated post-disaster is an urgent equity and public health problem.

### Rationale for Refinement
The general problem of emergency resource allocation is very broad and can include everything from food distribution to hospital planning. This makes it hard to break down into a focused, actionable plan. Narrowing it to housing assistance after flooding and hurricanes is helpful because renter/owner disparity is well-documented and hurricanes and flooding are the most common disaster types receiving federal aid. The availability of public datasets make this specific problem approachable for a data-driven solution.

### [**Press Release: Renters Left Behind After Floods!**](PressRelease.md)



## Domain Exposition

### Terminology

| Term / KPI | Definition |
|---|---|
| **FEMA IHP** | FEMA's Individuals and Households Program. The federal program that provides housing and other needs assistance to disaster survivors. |
| **Housing Assistance (HA)** | IHP sub-program covering temporary rental assistance, home repair grants, and home replacement funds after a disaster declaration. |
| **Valid Registrations** | Total number of households that submitted a complete, processable application for FEMA assistance in a given area. |
| **Approved for FEMA Assistance** | Boolean/count field indicating whether a household was deemed eligible and received any form of IHP award. |
| **Average FEMA HA Amount** | Mean dollar value of housing assistance awarded per approved applicant in a county/ZIP, across owners or renters separately. |
| **Disaster Declaration Number** | Unique FEMA identifier assigned to each presidentially declared disaster, used to filter dataset records to a specific event. |
| **Incident Type** | FEMA classification of the disaster (e.g. Hurricane, Flood, Severe Storm) used to filter records by disaster category. |
| **Approval Rate** | Percentage of valid registrants who were approved for assistance. |
| **CDBG-DR** | HUD's Community Development Block Grant. A Disaster Recovery program, a supplemental federal grant used for long-term housing rebuilding after major disasters. |
| **Social Vulnerability Index (SVI)** | CDC index scoring census tracts on 16 socioeconomic and demographic factors that affect a community's ability to recover from disasters. |



### Domain
This project lives in the domain of post-disaster housing recovery and its policy and resource allocation. It will use FEMA's OpenFEMA Housing Assistance datasets to examin how federal housing aid is distributed across tenure types after hurrican and flood events. Both datasets are structured at the county and ZIP code level and share comparable fields. The domain lies on an intersection of emergency management, housing policy, and data-driven equity analysis.

### Background Reading
[Project 2 Background Readings](https://1drv.ms/f/c/fbb5975494ef3dae/IgCPM-hb9pR3TaGLGrffvh_0Aa52Y874zWJSPDEkzBNX4uM?e=aJfCTl)

### Summary Table
| Title | Description | Link |
|---|---|---|
| How to Incorporate Equity in Post-Disaster Recovery Strategies | Summarizes a CDBG-DR report identifying five themes in grantee experiences: outreach, serving marginalized communities, types of recovery, administrative concerns, and households served. | [Link](https://1drv.ms/t/c/fbb5975494ef3dae/IQDHq-thLOLXSIKlXrJVRKSuAZ7w2gWJFM5qgIAfmUX8OeA?e=RLsY4J) |
| Evaluating Equity Efforts and Outcomes of CDBG-DR Funded Flood Resilience in Four Communities | HUD/Enterprise study on how CDBG-DR recipients experienced flooding disasters differently, finding key challenges in outreach, serving populations with specific needs, and overcoming administrative burdens. | [Link](https://1drv.ms/t/c/fbb5975494ef3dae/IQDLu0T_hzQiTbqIXu1KR4j_AUkaLQaveoH7JNQ_iQGdfnI?e=dRfha4) |
| Beyond Damages: Social Equity in Allocating Disaster Assistance | Examines how disaster assistance allocation intersects with social equity, arguing that damage assessments alone fail to capture the disproportionate impact on vulnerable populations. | [Link](https://1drv.ms/t/c/fbb5975494ef3dae/IQBq5U4HEEVlTIWGUO2tWt_IAf4VJF9RF7KzaKYcUH8321I?e=HnYezd) |
| Wildfire Recovery and Resilience Strategies for Resource-Constrained and Vulnerable Communities | Explores wildfire impacts on low-income families, older adults, people with disabilities, and rural residents, with recommendations for equitable support services. | [Link](https://1drv.ms/t/c/fbb5975494ef3dae/IQAzv3tMLVRZSpOw4a0IOGwCAeXapgZWktf53dYAy3Puvpc?e=dJGd2f) |
| FOURTEEN: Centering racial justice in the US emergency response framework | Argues that addressing disaster recovery disparities requires a more equitable approach to resource allocation that focuses on the needs of historically marginalized communities. | [Link](https://1drv.ms/t/c/fbb5975494ef3dae/IQCkx7z6kB2KQ6VizyWZUDhWAb82NHseQviKNpwxyzt3HCY?e=CbPRWR) |


## Data Creation

### Data Acquisition

Data was acquired through the official Federal Emergency Management Agency (FEMA) website. Both homeowner and renter data were found on the *OpenFEMA Dataset: Housing Assistance Program Data* webpage. From there, data was loaded into the `Pipeline.ipynb` Jupyter Notebook through each dataset's API endpoint, each in batches of 1000. This was done using the `requests` package to retrieve JSON responses, then converted into pandas DataFrames for analysis. Data was filtered to only include Hurricane/Flood disasters to scope the dataset to a single event for homeowner vs renter comparison.

### Code

|File| Description|Link|
|---|---|---|
|Pipeline.ipynb| Full data pipeline: loads in FEMA housing assistance data for homeowners and renters| [Link](https://github.com/kguzman24/Allocating-Emergency-Resources-Homeowners-Renters/blob/main/Pipeline.ipynb)|
|EPA.ipynb| Exploratory Jupyter Notebook to gather data needed for data summary, not necessarily analysis| [Link](https://github.com/kguzman24/Allocating-Emergency-Resources-Homeowners-Renters/blob/main/EPA.ipynb)|


### Rationale

The most notable decision was scoping the dataset to a single disaster event. This was done to ensure a controlled comparison between homeowner and renter resource allocation.Combining multiple disaster events of varying scale, geography, and severity could introduce variation that obscures patterns. Analyzing a single event allows for a more interpretable comparison under the same conditions. A second key decision was the selection of variables retained from the API query, specifically validRegistrations, approvedForFemaAssistance, and totalApprovedIhpAmount, which were chosen because they most directly capture both the demand for assistance and the financial outcomes of the allocation process, making them the most relevant features for evaluating equity between homeowners and renters.

### Bias Identification

Bias could be introduced in the data collection process in a couple different ways. First,  FEMA's housing assistance records only capture individuals who actively applied for aid, meaning those who did not or could not apply are absent from the dataset. This creates systematic underrepresentation of the most vulnerable populations such as undocumented residents, those without internet access, or individuals who are unaware of available assistance programs. Additionally, renters may be underrepresented relative to homeowners, as rental assistance programs may have different eligibility threholds and documentation requirements that could discourage or disqualify applicants. Finally, since the data reflects a single disaster event (4827), any patterns observed may not generalize across different hurricane or flood events, introducing selection bias into broader conclusions about resource allocation.

### Bias Mitigation

To mitigate the biases listed above, there could be transparency and rather than treating the dataset as a complete representation of all disaster-affected individuals, acknowledge the limitations of FEMA's application-based data. Through this, any interpretations of the conclusion will take note of underrepresented population that may be absent from records. Additionally, to avoid overgeneralizing findings, the analysis is intentionally scoped to disaster 4827 specifically, and no broad claims are made about homeowner vs renter resource allocation patterns across all hurricane or flood events. These measures don't eliminate underlying biases in the data, but they ensure that the analysis remains transparent about what the data can and can't tell us.

## Metadata

### Implicit Schema

The overall structure of the data is tabular, with each row representing a state level application and approval for housing assistance and each column capturing registration and metrics.


### Data Summary
|Statistic| Owners Dataset |Renters Dataset|
|---|---|---|
Total Records|1,409|1,457|
|Missing Values|0|0|
|Disaster Event| 4827|4827|
|States| NC, SC|NC,SC|
|Zipcodes|269 across both|269 across both|



### Data Dictionary
|Name|Data Type|Description| Example| Uncertainty|
|---|---|---|---|---|
|`state`|string|Two-letter postal abbreviation of the state where assistance was applied for| `NC`| 0 - No uncertainty as state is a fixed label assigned at registration and does not change.|
|`validRegistrations`|integer|Number of valid FEMA assistance registrations submitted in that state| `2`| .20 - Low uncertainty at the time the data was pulled. However, FEMA may update or invalidate registrations, meaning this could change if the dataset is queried at a later time.|
|`approvedForFEMAssistance`| integer| Number of registrations approved for FEMA housing assistance|`2`| 0.5 - Moderate uncertainty since approval statuses can be revised as FEMA processes appeals or conducts after the initial approval.|
|`totalApprovedIhpAmount`|float|Total dollar amount of approved Individual and Household Program (IHP) assistance| `8220.98`| .50 - Moderate uncertainty as disbursed amounts can be adjusted if approvals are appealed or supplemental assistance is granted.|








