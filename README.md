## 1. Scope the Project and Gather Data

#### Introduction
For this Udacity-provided, we have been provided with 4 data sources covering:
	- airport data on name, location, industry identification codes, and geographic elevation (source: https://datahub.io/core/airport-codes#data)
	- US demographic data summarising, for US cities, the number of people, males and females, veterans, and a breakdown by race (source: https://public.opendatasoft.com/explore/dataset/us-cities-demographics/export/)
	- world temperature data recording the average monthly temperature for cities across the world from the 1700s to 2013 (source: https://www.kaggle.com/datasets/berkeleyearth/climate-change-earth-surface-temperature-data)
	- I94 immigration data on international arrivals to the US, and includs visa types, some personal information, and the data and port of arrival (source: https://www.trade.gov/national-travel-and-tourism-office) NB: this dataset is broken down by month, with roughly 3 million entries per month. 

#### Use cases
We shall use these datasets to construct analytics tables which aid the analytic investigation of immigration patterns. The knowledge gained would benefit US cities administrators in:
	- extrapolating future population movements
	- understanding how demographic breakdown and location affects current populations
	- how climate affects demographic breakdown
	- understanding what might contribute to their immigrantion figures. 
It would also benefit potential US immigrants in identifying host cities with:
	- similar climates to their home country
	- particular demographic breakdowns, e.g., some people may prefer to relocate to more diverse cities. 
The latter of these two cases would likely require a more user-friendly front end - an app for example. We do not provide that here, but are able to test some queries as a proof of our concept. 


## 2. Explore and Assess the Data

To provide a reliable data product, we must explore and assess our data sources. We outline here some steps taken to prepare the data. 

Airport data:
	- contains 55075 rows
	- iata_code is missing for around 83% of the airports are null, so we shall drop this column
	- the iso_region column, which gives US state, contains entries of the form US-[state code]. We drop the 'US-' prefix to allow joining on city and state. 
	- we do not make use of data for specific airports, as immigration data is not provided at this granularity. Instead, we roll up to the municipality level (equivalent to 'city' in the other datasets below), which will allow us to join immigration data. 
	- for reasons explained below in the discussion of the temperature data, we reformat the latitude and longitude so that it matches the temperature data - e.g., -45.2525352 becomes 45.25S. This will allow us to join airport and temperature data. 

US demographic data:
	- contains 2891 rows
	- there are only 16 rows with any nulls, covering 8 relatively small cities, which we disclude as a cleaning step
	- for each city, there are between one and five rows
	- breakdown by ethnicity is provided in columns title 'race' and 'count', which results in the duplication of each city, along with their non-ethnic counts, between 1 and 5 times, depending on the number of ethnic groups. Duplication accunts for almost 80% of this table. We separate the race and count columns into a new table to substantially reduce this duplication
	- Vermont (VT) is missing from this dataset

Temperature data:
	- contains 8,599,212 columns
	- monthly temperature data is given for the period from 1743 to 2013. However, we note that the immigration data only covers 2016, so there is no time overlap between these two datasets. Since we are not able to compare time periods between these data sets, we opt to condense the temperature dataset using the most recent 10 years of data, which will be sufficient to identify the average monthly temperature. Hence, in the dataset taken forward, there will be 12 rows for each city, one for each month, which is the mean of the previous 10 years of temperature data for that month.
	- cities in the US are often not uniquely name, and this dataset does not the corresponding state/state_code for the cities. As such, we will not always be able to identify a city. As a workaround, we may make some progress by utilising latitude and longitude information to perform more ccurate joins. 

Immigration data:
	- contains 40,790,529 rows
	- some columns comprise a high proportion (>95%) of nulls, including 'occup' (occupation), 'entdepu' (), and 'insnum' (). We shall exclude these rows.
	- the processing port column ('194port') contains 109800 rows with 'XXX' entries, which indicate the port is unknown. As this would not help in later analyses, we disclude these rows. 
	- there are several columns that do not aid our use case, so will be discluded:
		- i94mode (mode of transport of arrival): does not aid use case
		- count: summary statistic
		- dtadfile: date added to I94 file
		- visapost: DoS which issued the visa
		- occup: mostly nulls
		- insnum: INS number
		- admnum: admission number
		- fltno: flight number
		- visatype: covered in i94visa

A final obseravtion we make which covers all of these datasets is cities occasionally have non-unique names (e.g., London, UK and London, Canada), so we take care to include country and US state where possible to avoid conflating like-named cities. 

The data-cleaning steps outlined above are presented in the file Data_exploration_and_assessment.ipynb



## 3. Define the Data Model

To assemble analytics tables from our data, we follow the below data model. 

[powerpoint diagram]

The use case we outline above centres around the immigration data, so we make this the core of our fact table. We construct the other tables as follows:
	- The 'dates' table contains the full date, as well as a breakdown by day, week etc. The month column here will be used to join immigration data to city temperature data. 
	- The port_codes table contains the US city and two-letter state code obtained from the I94_SAS_Labels_Descriptions.SAS file. 
	- The citres_codes table contains the citizen and resident codes to identify the immigrant's respective country of citizenship and resident, obtained from the I94_SAS_Labels_Descriptions.SAS file. 
	- The visa_code table contains the description of the 3 visa codes, obtained from the I94_SAS_Labels_Descriptions.SAS file. 
	- The demographics table contains demographic information on US cities. We require both the city and state code to join this table to others. 
	- The race table contains ethnic breakdown by city from the demographics data. 
	- the state table contains the state code and the corresponding full state name. 
	- The airports table contains information from the airports data which serves only to more accurately join the temperature table to other tables in the model.
	- The temperature table contains the monthly average temperature (from the most-recent 10 years of data) for cities across the world. 

To implement this data model, we use Spark running in a virtual workspace. 
We use PySpark to load, transform and build our data model tables, which we store in parquet format. 
The data is not too large to run using a single cluster, taking only a couple of minutes to process all of our data. 


## 4. Run ETL to Model the Data

The data pipeline is given in etl.py, with a data dictionary given in data_dictionary.txt. 
This pipeline load and processes the source data, completes validity checks, and saves them to parquet files, partitioned where appropriate. 



## 5. Scenarios
In addition to some sample queries included in the etl.py file, we consider here some scenarios for future development considerations and directions. 

- If data is increased by 100x.
One may envisage future functionality where we have immigration data that covers several years. We would use this data to gain insight into how the temperature change over several years affect immigration to a city. In this case, we would need to host and manage the data differently to handle larger volumes. 
For example, if the volume of data was increased by 100-fold, we would consider hosting our product on a public cloud provider, where we could more easily scale storage and computational resources. We could increase clusters sizes more readily by using Amazon Redshift. 

- If the pipelines were run on a daily basis by 7am.
If we needed to run repetitive tasks at regular time intervals, a clear choice for further development would be Airflow, which would enable automatic run retries and email notification. 

- If the database needed to be accessed by 100+ people.
If this product gains good engagement with city administrators and would-be immigrants, we could support them by using AWS IAM role and group configurations for analysts, and host a front end on AWS which is fed by an AWS resource which is optimised for reading to handle multiple concurrent requests. 
