---
title: "IoT data warehouses 3<br/> Pipelines"
date: 2022-08-15T15:00:00-00:00
categories:
  - Data
  - IoT
tags:
  - GCP
  - BigQuery
  - Cloud Run
  - App Engine
  - SQL
  - Python
---

We've managed to streamline the process of reading data from our data warehouse, how can we also make writing to it as painless as possible? While connecting to external spreadsheets and tables is all well and good, to ingest a high volume of live data we need pipelines. Here are some pipelines we can set up using GCP services.

## Cloud Run
Webhooks are a popular method of transferring data between servers on the web, due to their simplicity and non-reliance on shared state. Cloud Run gives us a way to ingest data from other web services into our data warehouse via webhooks. It provides a serverless environment for containers, and exposes these containers to other GCP services or to the internet via a public URL. The benefit of serverless is that you are only billed for active container time (such as when a request is being handled), whereas a servered approach would require a VM running 24/7.

In regards to our LoraWAN sensors, we need one additional component to get the data from our sensors onto the internet. This is called an LNS (LoraWAN Network Server) and there are many free and open source options available such as TTN. TTN supports a webhook integration, so all we need to is put the URL of our Cloud Run service into TTN, and the data will be forwarded automatically.

Within our container code, we can use the BigQuery Python API to get data into BigQuery. As you may remember, our data analysts are already using this API to run queries on our data warehouse. However, in this case we are using another feature of the API - streaming inserts - to send data in the opposite direction. We could potentially also run queries within the same container (to clean / transform incoming data using our metadata, for example), but there a few of reasons why we should avoid this:

1. It's redundant since we alredy have the materialised view to carry out such operations for us
2. Our quota usage will increase
3. There are risks associated with discarding data - even erroneous data - before it reaches our data warehouse. Such data could be critical in diagnosing sensor / pipeline issues for example

A very basic Cloud Run service written in Python could use the Flask framework to handle incoming webhook requests. It is possible to ingest data from multiple sources using a single container this way, as we can specify multiple endpoints in Flask. However, for the sake of modularity it is generally a better idea to deploy one Cloud Run service for each webhook.

```python

from flask import Flask, request, jsonify
from google.cloud import bigquery

app = Flask(__name__)

client = bigquery.Client()
bq_table = client.get_table('Sensor_data')

@app.route("/", methods=["POST"])
def main():

	content = request.get_json()

	//
	// Carry out transformations here if neccessary
	//
	
	errors = client.insert_rows_json(bq_table, content)
	if errors == []:
		print("New rows have been added.")
	else:
		print("Encountered errors while inserting rows: {}".format(errors))

app.run(debug=True, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))

```

Once deployed, GCP associates a unique URL with this service and any POST requests made to it will reach the main() function. The Cloud Run (internal) service can be made more secure by requiring authentication, and this is typically achieved by sharing service account credentials with the calling (external) service.

## Mail
Suppose we receve some data in the form of email attachments, how can we get this data into BigQuery without needing to manually download and reupload it ourselves? Cloud Run is not applicable in this instance, however another GCP service - App Engine - can come to our aid. App Engine supports the Mail API which allows us to receive emails and process the attachments programmatically in Python. We still use Flask to handle requests, however this time the trigger is an email and not a webhook. The target email address is determined by the App Engine service name and GCP project id: RECIPIENT@SERVICE_NAME-dot-PROJECT_ID.appspotmail.com (the RECIPIENT does not matter but can be used for additional filtering). The following code will handle attachments in the form of multiple zipped csvs:

```python

from flask import Flask, request
from google.appengine.api import mail
from google.appengine.api import wrap_wsgi_app
from google.cloud import storage
from google.cloud import bigquery
import io
import pandas as pd
from zipfile import ZipFile, is_zipfile

app = Flask(__name__)

@app.route("/_ah/mail/<path>", methods=["POST"])
def receive_mail(path):

	message = mail.InboundEmailMessage(request.get_data())

	# extract message info
	recipient = message.to.split('@')[0]

	# for emails with no attachments, print the contents
	if not hasattr(message, 'attachments'):

		print(f"Received email for {recipient} at {message.date} from {message.sender}")

		for content_type, payload in message.bodies("text/plain"):
			print(f"Text/plain body: {payload.decode()}")
			break

	else:

		for filename, filecontent in message.attachments:

			if '.zip' in filename.casefold():

				with ZipFile(io.BytesIO(filecontent.decode())) as myzip:

					for unzippedfilename in myzip.namelist():
						unzippedfilecontent = myzip.read(unzippedfilename)

						if '.csv' in unzippedfilename.casefold():
							send_to_data_warehouse(unzippedfilename, unzippedfilecontent)

						else:
							print('no csv files found')

			else:
				print('no zip files found')

	return "OK", 200

```

How to implement our send_to_data_warehouse function? There are a couple of options available: 

1. If the attachments are csvs (like in the above example) or json files, we can upload them straight to Google Cloud Storage and add them to BigQuery as an external data source. This makes our App Engine code very simple, but potentially give us headaches further down the line. External data sources are not supported in materialised views, so this will make it harder to optimise queries involving this data (However, you can bypass this limitation if you upgrade to {BigLake tables with Hive partioning}(https://cloud.google.com/bigquery/docs/create-cloud-storage-table-biglake){:target="_blank"}).

```python

def send_to_data_warehouse(uri, unzippedfilecontent):

	print(f'writing file: {uri}')

	storage_client = storage.Client()
	bucket = storage_client.bucket('my_gcs_bucket')
	blob = bucket.blob(uri)
	blob.content_type = 'text/csv'

	with blob.open("w") as f:
	f.write(unzippedfilecontent.decode())

```

2. We can use streaming inserts to pipe the data into a BigQuery native table, allowing us to optimise our queries using materialised views. However, this comes at the cost of making our App Engine code more complicated since we need to transform the data before we can stream it.

```python

table_schema = {

	names : [
		
		'device_id',
		'timestamp',
		'temperature',
		'humidity',
	],
	types : {
		
		'device_id' : str,
		'timestamp' : timestamp,
		'temperature' : float,
		'humidity' : float
	}	
}

def send_to_data_warehouse(uri, unzippedfilecontent):

	print(f'streaming to table: {table}')

	client = bigquery.Client()
	bq_table = client.get_table('Sensor_data')
	
	df = pd.read_csv(io.BytesIO(unzippedfilecontent), header=1, names=table_schema['names'], dtype=table_schema['types'])

	bigquery_client.insert_rows_from_dataframe(bq_table, df)

```

## PubSub
GCP also has a BigQuery connector for PubSub, which is a service for publishing and subscribing to message queues. PubSub is primarily intended for internal messaging within GCP, but there are some external services which can publish directly to PubSub queues, making for a very simple and secure data pipeline (since there's no need to expose a public URL unlike with a webhook). The only difficulty in setting this up is the message schema must match the BigQuery schema, so there might be some adjustments to be made in BigQuery before data can be ingested. A typical JSON message schema could have sensor values nested within a payload object like this:

```json

{
	device_id : aaa,
	timestamp : bbb,
	payload : {
	
		temperature : xxx,
		humidity : yyy,
	}
}

We would need to update our Sensor_data table accordingly:

|---
| Sensor_data
|-
| device_id
| timestamp
| payload

Our sensor values are now hidden in a JSON string, so how can we extract the raw values for our queries? BigQuery has some JSON functions which can help us out, and JSON_EXTRACT_SCALAR is just what we're looking for:

```sql

SELECT device_id, timestamp,
JSON_EXTRACT_SCALAR(payload, '$.temperature') as temperature,
JSON_EXTRACT_SCALAR(payload, '$.humidity') as humidity
FROM Sensor_data

```
