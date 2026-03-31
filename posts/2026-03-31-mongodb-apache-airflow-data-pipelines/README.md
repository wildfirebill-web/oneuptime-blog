# How to Use MongoDB with Apache Airflow for Data Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Apache Airflow, Pipeline, ETL, Orchestration

Description: Learn how to build MongoDB data pipelines orchestrated by Apache Airflow using the MongoHook and custom operators to extract, transform, and load data on a schedule.

---

Apache Airflow orchestrates MongoDB data pipelines by scheduling DAGs (directed acyclic graphs) that extract, transform, and load data between MongoDB and other systems. The `apache-airflow-providers-mongo` package provides a `MongoHook` for connecting to MongoDB from Airflow tasks.

## Install the Provider

```bash
pip install apache-airflow-providers-mongo
```

## Configure a MongoDB Connection in Airflow

In the Airflow UI, go to **Admin > Connections** and create a connection:

- **Conn ID**: `mongo_default`
- **Conn Type**: `MongoDB`
- **Host**: `localhost`
- **Port**: `27017`
- **Schema**: `salesdb`

Or set it via environment variable:

```bash
export AIRFLOW_CONN_MONGO_DEFAULT='mongodb://localhost:27017/salesdb'
```

## Basic ETL DAG

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.mongo.hooks.mongo import MongoHook
import pandas as pd

default_args = {
    "owner": "data-team",
    "retries": 2,
    "retry_delay": timedelta(minutes=5)
}

def extract_orders(**context):
    hook = MongoHook(mongo_conn_id="mongo_default")
    client = hook.get_conn()
    db = client["salesdb"]

    pipeline = [
        { "$match": { "status": "completed", "createdAt": { "$gte": datetime(2025, 1, 1) } } },
        { "$project": { "_id": 0, "orderId": 1, "amount": 1, "category": 1, "createdAt": 1 } }
    ]

    records = list(db.orders.aggregate(pipeline))
    context["ti"].xcom_push(key="record_count", value=len(records))
    return records

def transform_orders(**context):
    records = context["ti"].xcom_pull(task_ids="extract_orders")
    df = pd.DataFrame(records)
    df["revenue_band"] = pd.cut(df["amount"], bins=[0, 50, 200, 1000, float("inf")],
                                labels=["low", "medium", "high", "premium"])
    return df.to_dict("records")

def load_summary(**context):
    records = context["ti"].xcom_pull(task_ids="transform_orders")
    hook = MongoHook(mongo_conn_id="mongo_default")
    client = hook.get_conn()
    db = client["salesdb"]
    db.order_summary.drop()
    db.order_summary.insert_many(records)
    print(f"Loaded {len(records)} documents")

with DAG(
    dag_id="mongodb_etl_pipeline",
    default_args=default_args,
    schedule_interval="@daily",
    start_date=datetime(2025, 1, 1),
    catchup=False,
    tags=["mongodb", "etl"]
) as dag:

    extract = PythonOperator(task_id="extract_orders", python_callable=extract_orders)
    transform = PythonOperator(task_id="transform_orders", python_callable=transform_orders)
    load = PythonOperator(task_id="load_summary", python_callable=load_summary)

    extract >> transform >> load
```

## Use MongoHook Directly for Simple Queries

```python
from airflow.providers.mongo.hooks.mongo import MongoHook

def count_new_users(**context):
    hook = MongoHook(mongo_conn_id="mongo_default")
    client = hook.get_conn()
    db = client["salesdb"]

    yesterday = datetime.utcnow() - timedelta(days=1)
    count = db.users.count_documents({ "createdAt": { "$gte": yesterday } })
    print(f"New users yesterday: {count}")
    return count
```

## Add Data Quality Checks

```python
from airflow.operators.python import BranchPythonOperator

def check_record_count(**context):
    count = context["ti"].xcom_pull(task_ids="extract_orders", key="record_count")
    if count < 100:
        return "send_alert"
    return "transform_orders"

quality_check = BranchPythonOperator(
    task_id="quality_check",
    python_callable=check_record_count
)
```

## Summary

Apache Airflow with MongoDB uses `MongoHook` for database connections and PythonOperator tasks for ETL logic. Structure your DAGs as extract, transform, and load steps, add data quality checks using BranchPythonOperator, and schedule daily or hourly runs for automated reporting pipelines. Store Airflow connections in the UI or environment variables to keep credentials out of DAG code.
