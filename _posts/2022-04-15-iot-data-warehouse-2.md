---
title: "IoT data warehouse 2<br/> Schemas"
date: 2022-04-15T15:00:00-00:00
categories:
  - Data
  - IoT
tags:
  - GCP
  - BigQuery
---

Last time we discussed how to implement a data warehouse for a typical IoT application. While our current solution is usable, we aren't yet leveraging the true potential of our warehouse to make downstream analysis as painless as possible. How can we:

1. Make our data easy to interpret?
2. Ensure queries are fast and cost-effective? 
3. Limit the amount of SQL our data analysts need to write?
4. Clean the data? 

Luckily, BigQuery has a few features which can help us out.

### Schema optimisation
Recall our schema from before: 

|---
| Metadata | Sensor_data
|-|
| device_id | device_id
| sensor_type | timestamp
| manufacturer | temperature
| part_number | humidity
| connection_key | 
| longitude | 
| latitude |

From a data analysts perspective, our tables contains a lot of redundant information since they only care about a handful of values within them. We can make our schema more flexible by introducing an additional level of abstraction - assets. We conceptualise assets as sensors with all the hardware aspects stripped away, making them pure data sources. By defining assets alongside sensors, we gain a few advantages over our prior schema:

1. We can link assets to physical things in the real world (vehicles for example) or to a fixed location, which makes interpreting the data a lot easier
2. By allowing many-to-one relationships betweens sensors and assets, a single asset can be associated with many types of data
3. We can easily move / swap sensors while keeping assets fixed, which is useful if you need to perform sensor maintenance for example

Our revised schema looks like this:

|---
| Assets | Devices | Sensor_data
|-|-|
| asset_id | device_id | device_id
| name | sensor_type | timestamp
| longitude | manufacturer | temperature
| latitude  | part_number | humidity
| | connection_key |

The Assets table now contains all our high level information, including a 'name' field to easily identify our data source. There is one piece missing from this schema - we are currently unable to join all these tables together since we lack a join key. We can overcome this using an ancillary table like this:

|---
| Assignments
|-
| timestamp
| device_id
| asset_id

Anytime we assign a device to an asset, we record it in this table along with the time we wish the assignment to take effect. How can we use this to join our Assets and Devices tables together? Simply put, we need to calculate time windows (start and end times) using our assignments, and compare these to the sample timestamps during our table join. The LEAD operator - one of the many analytic functions available in BigQuery - gives us our windows by grouping and sorting our assignments. Thus, our query ends up looking like this:

```SQL

-- We need to specify the time period in which all our data resides
DECLARE start_time DEFAULT "2000-01-01"
DECLARE end_time DEFAULT "2030-12-31"

-- Convert our assignments into windows
WITH Windows AS (

	SELECT 
	
		IF(t < start_time or t is null, start_time, t) as window_start,
		IF(lead_t > end_time or lead_t is null, end_time, lead_t) as window_end, 
		device_id,
		asset_id

	FROM (
	
		SELECT *,
		LEAD(timestamp) OVER (PARTITION BY device_id ORDER BY timestamp) as lead_t
		FROM Assignments
	)

)

-- First join adds our assigned asset_id to each row of sensor data
SELECT * FROM Windows w JOIN Sensor_data s
ON s.device_id = w.device_id
AND s.timestamp BETWEEN window_start and window_end

-- Second joins add our columns from Assets
JOIN Assets
USING (asset_id)

```

We may have improved the schema to make it easier to interpret, however our data scientists won't be thanking us just yet given the amount of extra SQL we're making them write. BigQuery's next feature will help us out there.

### Materialised views
Materialised views are like snapshots of a SQL query, where the results are refreshed automatically (when the underlying tables are updated, or on a regular schedule) to maintain data freshness. Once a view is defined, it can be queried just like a regular table. They are particularly useful if you are expecting a lot of similar looking queries. Materialised views perform resource intensive operations - such as joins - ahead of time, making our interactive queries more responsive and easier to write. 

Despite their obvious benefits, materialised views come with a few prequisites. For one, they only support BigQuery native tables as inputs, so for our IoT application we need to move our current spreadsheet-based metadata over to this format. [Here is a good guide for how to do this using an Apps Script](https://towardsdatascience.com/google-sheets-to-google-bigquery-move-your-data-a5fa9f4e9e5d){:target="_blank"}. Secondly, analytic functions are not supported in the definition of a materialised view, and so our time windows must be built ahead of time and inserted into another native table. A good way to do this might be using a scheduled procedure to update the windows periodically or on demand:

```SQL

BEGIN

-- First delete our existing windows
DELETE Windows
WHERE window_start IS NOT NULL;

-- Recalculate and insert new windows
INSERT Windows

SELECT 

	IF(t < start_time or t is null, start_time, t) as window_start,
	IF(lead_t > end_time or lead_t is null, end_time, lead_t) as window_end, 
	device_id,
	asset_id

FROM (

	SELECT *,
	LEAD(timestamp) OVER (PARTITION BY device_id ORDER BY timestamp) as lead_t
	FROM Assignments
);

END

```

With all this taken into consideration, we define our materialised view like this:

```SQL

CREATE OR REPLACE MATERIALIZED VIEW Sensor_view
PARTITION BY DATE(timestamp)
CLUSTER BY device_id
AS (

-- First join adds our assigned asset_id to each row of sensor data
SELECT DISTINCT * FROM Windows w JOIN Sensor_data s
ON s.device_id = w.device_id
AND s.timestamp BETWEEN window_start and window_end

-- Second joins add our columns from Assets
JOIN Assets
USING (asset_id)

-- Apply some filtering
WHERE temperature IS NOT NULL
AND humidity IS NOT NULL
AND temperature BETWEEN -100 AND 100
AND humidity BETWEEN 0 AND 100

)

```

Here we also use partitioning and clustering to further optimise the view, by telling BigQuery to colocate similiar data. We can also perform some data cleaning by filtering rows with null or invalid columns, and use the DISTINCT keyword to remove duplicates.





