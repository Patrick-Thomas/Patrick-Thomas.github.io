---
title: "IoT data warehouses"
date: 2022-01-15T15:00:00-00:00
categories:
  - Data
  - IoT
tags:
  - GCP
  - BigQuery
  - SQL
  - Python
---

The number of IoT devices available on the market has exploded in the last decade, and this has led to a monitoring and optimisation boom in many industries. However, collecting a high volume of diverse data comes with additional challenges, one of which is how to organise and extract insights from this data. Thankfully, the tech giants have known about these problems for a lot longer than most of us, and have some solutions we can make use of. One of these is the data warehouse.

## What is a data warehouse?
A data warehouse can be thought of as an evolution of a database. Databases typically store data using one type of object - such as a table, key-value pair or blob - however data warehouses provide the flexibility to mix and match the types of objects used, while still allowing users to access the data in a homogeneous fashion (using a single SQL statement, for example). This greatly speeds up analysis of the underlying data. 

Introducing this flexibility adds some additional processing overhead, so data warehouses are not renowned for having millisecond-level query response times. However, data warehouses are usually built on top of cloud infrastructure, which means they can scale horizontally to accommodate different workloads. This not only makes them capable of handling huge volumes of data (petabytes), but provides other benefits such as failover protection which are trickier to implement for on-premises databases.

The 3 big cloud providers - AWS (Amazon Web Services), GCP (Google Cloud Platform) and Microsoft Azure - all have their own data warehouses. These are Redshift for AWS, BigQuery for GCP, and Synapse for Azure. There are also standalone offerings available such as Snowflake. My experience is with BigQuery, but most principles are applicable to data warehouses as a whole.

## A real world example
Suppose we have a large quantity of LoraWAN sensors (long range sensors optimised for low power, low data rate applications) deployed across multiple UK ports. Our database schema looks like this:

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

For simplicity, we can store the metadata in a spreadsheet to make it easy to read and modify. To import this data into our data warehouse of choice - BigQuery - we can make use of the Google Sheets connector. Our live sensor data can be piped directly into a native BigQuery table (a process I'll describe later), and with both sets of data now in our warehouse, we can run SQL queries such as the following:

```sql

SELECT device_id, AVG(temperature) as average_temperature FROM Sensor_data
JOIN Metadata USING (device_id)
JOIN `bigquery-public-data.geo_international_ports.world_port_index`
ON ST_DWITHIN(ST_GEOGPOINT(longitude, latitude), ST_GEOGPOINT(port_longitude, port_latitude), 5000)

WHERE DATE(timestamp) = '2023-03-01'
AND port_name = 'FELIXSTOWE'

GROUP BY device_id 

```

This gives us the average temperature recorded by each of our sensors within a 5km radius of Felixstowe port on the 1st March 2023. Here we make use of a public dataset - geo_international_ports - which is just one of the many datasets available for free on BigQuery. By joining this public data with our Metadata using the geography functions ST_DWITHIN and ST_GEOGPOINT, we can associate each of our sensors with it's nearest port, and thus use it as a filter in our WHERE clause.

### Python API
To allow our data analysts to use their existing pandas workflow, we can leverage the BigQuery Python API to import data directly into a Jupyter notebook or any regular script. Once the google-cloud-bigquery package is imported and the user is authenticated (they require the appropriate IAM permissions to do this, a topic for another day), they can submit query jobs and get data back as a pandas dataframe. Here's an example script which uses the query from before, with our filter parameters added via formatting:

```python

from google.cloud import bigquery

client = bigquery.Client()

date = '2023-03-01'
port_name = 'FELIXSTOWE'

query_string = f'''

	SELECT device_id, AVG(temperature) as average_temperature FROM Metadata 
	JOIN Sensor_data USING (device_id)
	JOIN `bigquery-public-data.geo_international_ports.world_port_index`
	ON ST_DWITHIN(ST_GEOGPOINT(longitude, latitude), ST_GEOGPOINT(port_longitude, port_latitude), 5000)

	WHERE DATE(timestamp) = {date}
	AND port_name = {port_name}

	GROUP BY device_id 

'''

df = client.query(query_string).to_dataframe()

```

