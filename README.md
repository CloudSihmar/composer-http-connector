# Using the HTTP Operator in Google Cloud Composer

## Introduction

In this use case, we will demonstrate how to use the HTTP Operator in Google Cloud Composer (an implementation of Apache Airflow) to trigger an external HTTP endpoint. This can be particularly useful for integrating with external APIs, webhooks, or microservices as part of your data pipeline.

## Prerequisites

1. Google Cloud Platform (GCP) account.
2. Google Cloud Composer environment set up.
3. Access to an HTTP endpoint to trigger.

## Steps

1. **Create a New DAG**: Create a new Directed Acyclic Graph (DAG) in your Composer environment.
2. **Define the HTTP Operator**: Use the `SimpleHttpOperator` to define the HTTP request.
3. **Set Dependencies**: Define the order of tasks in your DAG.

## Example

### Create a new DAG file

Create a new file named `dag-http.py` in the `dags` directory of your Composer environment.

```python
"""Demo HTTP operators"""
from __future__ import annotations

import json
import os
from datetime import datetime

from airflow import DAG
from airflow.providers.http.operators.http import SimpleHttpOperator
from airflow.providers.http.sensors.http import HttpSensor
from airflow.operators.python import PythonOperator
from google.cloud import storage

# Dag name
DAG_ID = "demo_http_operator_to_gcs"

# ths python funcgion writes data from Xcom to GCS byucket as a JSON file
def WriteToGcs(ti):
    data = ti.xcom_pull(task_ids=['get_http_data'])
    bucket_name = 'sandeep-apache'
    destination_blob_name = 'data.json'
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(destination_blob_name)

    blob.upload_from_string(str(data))

    print(
        f"{destination_blob_name} with contents uploaded to {bucket_name}."
    )
# DAG definitions with all required params
dag = DAG(
    DAG_ID,
    default_args={"retries": 1},
    tags=["example"],
    start_date=datetime(2023, 4, 26),
    catchup=False,
)

# Task to get data from given HTTP end point
get_http_data = SimpleHttpOperator(
    task_id="get_http_data",
    http_conn_id="my_http_connection",
    method="GET",
    endpoint="/query?function=TIME_SERIES_DAILY&symbol=IBM&apikey=demo",
    response_filter = lambda response : json.loads(response.text),
    dag=dag
)
# Task to write data from Xcom to GCS bucket
write_data_to_gcs = PythonOperator(
    task_id = 'write_data_to_gcs',
    python_callable = WriteToGcs
)
# Task dependency set
get_http_data >> write_data_to_gcs
```

## HTTP Connection

Ensure you have an HTTP connection configured in your Airflow connections. You can set it up via the Airflow UI:

- Go to the Airflow web server UI.
- Navigate to Admin -> Connections.
- Click on + (Add a new record).
- Set Conn Id to my_http_connection.
- Set Conn Type to HTTP.
- Provide the host and other required details for your HTTP endpoint.
