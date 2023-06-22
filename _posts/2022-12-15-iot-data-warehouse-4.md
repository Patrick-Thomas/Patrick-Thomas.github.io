---
title: "IoT data warehouse 4<br/> Metrics"
date: 2022-12-03T15:00:00-00:00
categories:
  - Data
  - IoT
tags:
  - GCP
  - BigQuery
  - Cloud Run
  - Python
---

Our data warehouse now contains a diverse set of data which we can query with relative ease. Churning through raw data can be taxing however, so we typically need to calculate secondary values - or metrics - to make the key insights more accessible for our less data-savvy colleagues and clients. Here are a few ways of calculating such metrics inside GCP

## Scheduled queries
It's possible to calculate metrics from within BigQuery itself using scheduled queries. A scheduled query simply appends or inserts query results into a table of your choice on a given schedule. The query run time and date are accessible within the query environment via the parameters run_time and run_date, which makes calculating hourly averages trivial:

```SQL

SELECT device_id, AVG(temperature) as average_temperature FROM Sensor_data

WHERE timestamp BETWEEN TIMESTAMP_SUB(@run_time, INTERVAL 1 HOUR)
GROUP BY device_id 

```

Here the TIMESTAMP_SUB function allows us to only include data from the past hour in our calculation. BigQuery scheduled queries also support backfill, which is very useful if you want to calculate metrics for historical data. The preceding metric can be backfilled easily, since there is no 'overlap' between the calculated values. However, this is not always the case. Suppose we want to keep track of the total number of samples we've received from each sensor, and have the the value updated each day. The scheduled query for this metric would look like this:

```SQL

-- Get the latest count for each sensor from our existing metric table (Device_total_samples_metric)
WITH previous AS (

	SELECT count_total as count_previous, device_id
	
	FROM (
	
		RANK() OVER (PARTITION BY deviceId ORDER BY current_date DESC), count_total, device_id
		FROM Device_total_samples_metric
	)
	
	WHERE my_rank = 1
),

-- Get the number of samples from today by counting rows for each sensor
WITH current AS (

	SELECT COUNT(*) as count_today, device_id, @run_date as current_date 
	FROM Sensor_data
	WHERE timestamp BETWEEN TIMESTAMP_SUB(@run_date, INTERVAL 1 DAY)
	GROUP BY device_id
)

-- Calculate the new total
SELECT count_previous + count_today as count_total, device_id, current_date
FROM previous JOIN current USING (device_id)

```
In this case, performing a standard backfill would produce a discontinuity at the intersection of new and historic values. Instead, we can write a procedure to delete and recalculate all our values:

```SQL

BEGIN

-- First delete our existing metric values
DELETE Device_total_samples_metric
WHERE current_date IS NOT NULL;

-- Recalculate and insert new metrics
INSERT Device_total_samples_metric

-- Get the daily sample count for each sensor up to the current date.
-- TIMESTAMP_TRUNC ensures we don't count twice for the current day,
-- since we assume our scheduled query is going to run at the end of today
WITH count_daily AS (

	SELECT COUNT(*) as count_samples, device_id, DATE(timestamp) as ts_date
	FROM Sensor_data
	WHERE timestamp < TIMESTAMP_TRUNC(CURRENT_TIMESTAMP(), DAY)
	GROUP BY device_id, DATE(timestamp)
)

-- Use the analytic SUM function to calculate a running total
SELECT SUM(count_samples) OVER (PARTITION BY device_id ORDER BY ts_date) as count_total, device_id, ts_date as current_date
FROM count_daily

```

Certain metrics are even trickier to work with. For example, what if we want to apply an exponential filter to our temperature data? Analytic functions - such as SUM in the last example - let us carry out certain operations row-by-row, but there is no equivalent function for exponential filtering. Such operations are difficult to perform in pure SQL due to their 'non-vectorisable' nature. Therefore - to give ourselves more flexibility - we can write our own UDF (user defined function) using a non-SQL dialect. Currently, BigQuery only supports javascript (and SQL) UDFs, so that's what we'll use:

```SQL

CREATE OR REPLACE FUNCTION exponential_filter (input_array ARRAY<STRUCT<timestamp TIMESTAMP, value FLOAT64>>, alpha FLOAT64)
RETURNS ARRAY<STRUCT<timestamp TIMESTAMP, value FLOAT64>>
LANGUAGE js
AS r'''

// Constrain alpha between 0 and 1
alpha = Math.max(0, Math.min(1, alpha))

// loop through the input array and apply the exponential filter.
// There is no need to build a separate output array so we edit
// the input array in place to save memory
for (let i = 1; i < input_array.length; i++) {

	input_array[i]['value'] = alpha*input_array[i]['value'] + (1 - alpha)*input_array[i-1]['value']))
}

// We shift the array to remove the first element, because the
// first element already exists in our metric table and we don't
// want any duplicates
input_array.shift()

return input_array

''';

```

To use this function in our metric calculations, we need to convert our rows of temperature data into an array, and this is acheived using the ARRAY_AGG function. Once we've transformed our array using the exponential filter, we then need to use the 'inverse' of the ARRAY_AGG function - the UNNEST operator - to get our new rows. How do we ensure the filter runs for each of our sensors? We could include logic in our UDF to split our input array by device_id, but this approach is unnecessarily complex. Instead, we can use a procedural FOR loop in BigQuery to do the split. Such loops allow you to execute a set of SQL statements for each row in a table, similar to the forEach method in javascript. Here is what our scheduled query looks like:

```SQL

BEGIN

-- Choose alpha value for the exponential filter
DECLARE alpha DEFAULT 0.9

-- Get all of our temperature sensors
FOR record IN (SELECT * FROM Metadata WHERE sensor_type = 'temperature')

-- Run this for each device
DO

	-- Insert new metrics
	INSERT Temperature_filtered_metric

		SELECT timestamp, temperature_filtered, record.device_id, 
		
		-- Unnest our transformed array as a table
		FROM UNNEST (( 
		
			-- Scalar subquery only selects a single column and row containing our filtered array
			SELECT exponential_filter (ARRAY_AGG(STRUCT(timestamp, temperature)), alpha)
			
			-- Input rows for our filter function
			FROM (
			
				-- Rows from the past hour
				SELECT timestamp, temperature
				FROM Sensor_data
				WHERE device_id = record.device_id
				AND timestamp > TIMESTAMP_SUB(@run_time, INTERVAL 1 HOUR)
				
				UNION ALL
				
				-- Last row from the previous batch
				SELECT timestamp, temperature FROM (
				
					SELECT RANK() OVER (ORDER BY timestamp DESC) as my_rank, timestamp, temperature_filtered as temperature
					FROM Temperature_filtered_metric
					WHERE device_id = record.device_id
				)
				WHERE my_rank = 1
				
				-- We need to order our samples before we filter them
				ORDER BY timestamp
			)
		));

END FOR;
		
END

```

And here is our backfill calculation; we'd run this to filter all our historic data up to this point:

```SQL

BEGIN

-- Choose alpha value for the exponential filter
DECLARE alpha DEFAULT 0.9

-- First delete our existing metric values
DELETE Temperature_filtered_metric
WHERE current_date IS NOT NULL;

-- Get all of our temperature sensors
FOR record IN (SELECT * FROM Metadata WHERE sensor_type = 'temperature')

-- Run this for each device
DO

	-- Recalculate and insert new metrics
	INSERT Temperature_filtered_metric

		SELECT timestamp, temperature_filtered, record.device_id, 
		
		-- Unnest our transformed array as a table
		FROM UNNEST (( 
		
			-- Scalar subquery only selects a single column and row containing our filtered array
			SELECT exponential_filter (ARRAY_AGG(STRUCT(timestamp, temperature)), alpha)
			
			-- Input rows for our filter function
			FROM (
			
				-- All rows up to now, excluding those within the current hour
				SELECT timestamp, temperature
				FROM Sensor_data
				WHERE device_id = record.device_id
				AND timestamp < TIMESTAMP_TRUNC(CURRENT_TIMESTAMP(), HOUR)
				
				-- We need to order our samples before we filter them
				ORDER BY timestamp
			)
		));

END FOR;
		
END

```

## Cloud Run jobs
If we want to give our data analysts the ability to write metrics in python, we can leverage Cloud Run containers. Since these containers do not need to be accessible over the internet - contrary to our real time data pipelines - we can use a Cloud Run 'job' in place of a 'service'. Therefore, we don't need to use a web framework such as Flask to handle requests, and our code can be a lot simpler. We trigger the job using another GCP service such as Cloud Scheduler.

We will need to make use of the BigQuery Python API in our code - firstly to query our raw data, and then to insert our new metric values. Below is a python script we could use to calculate hourly average temperatures:

```python

from google.cloud import bigquery
import pandas as pd
import numpy as np
from datetime import datetime, timezone

client = bigquery.Client()
bq_table = client.get_table('Temperature_average_metric')

metric_rows = []

current_time = int(datetime.now(timezone.utc).timestamp())

query_string = f'''

	SELECT *
	FROM Sensor_data
	WHERE timestamp > TIMESTAMP_SUB(TIMESTAMP_SECONDS({current_time}), INTERVAL 1 HOUR)

'''

# obtain a dataframe containing raw BigQuery data
df = client.query(query_string).to_dataframe()

# split our data by device_id and create a metric row for each
df_split = pd.groupby(pd.Grouper(key='device_id', axis=0))

for device_id, df in df_split:

	avg = np.average(df.temperature)
	
	metric_rows.append
	({
		'timestamp': current_time,
		 'device_id': device_id,
		 'average_temperature': avg
	})

# construct a new dataframe and stream our metric rows to BigQuery
metrics = pd.DataFrame(metrics_rows)
errors = client.insert_rows_from_dataframe(bq_table, metric)

if errors == []:
	print("New rows have been added.")
else:
	print("Encountered errors while inserting rows: {}".format(errors))
	
```

Since we always need to write a SQL query to retrieve our raw data, it's possible to do part of the metric calculation in the initial query, and only use python to finalise our values (or to perform non-vectorisable operations such as exponential filtering). This gives our data scientists a lot of flexibility.

To run a backfill using this method, we need to write a separate python job.







