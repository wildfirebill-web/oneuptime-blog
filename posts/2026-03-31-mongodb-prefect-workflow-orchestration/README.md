# How to Use MongoDB with Prefect for Workflow Orchestration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Prefect, Workflow, Data Pipeline, Python

Description: Learn how to build Prefect flows that read from and write to MongoDB, with retry logic and observability for reliable data pipeline orchestration.

---

## Why Prefect with MongoDB?

Prefect is a modern Python workflow orchestration tool that wraps functions with retry policies, logging, and observability. Combining it with MongoDB lets you build resilient pipelines that extract, transform, and load data between MongoDB and other systems.

## Installing Dependencies

```bash
pip install prefect pymongo
```

## Connecting to MongoDB in a Prefect Task

```python
from prefect import flow, task
from pymongo import MongoClient

MONGO_URI = "mongodb+srv://user:pass@cluster.mongodb.net/"

@task(retries=3, retry_delay_seconds=10)
def read_orders(db_name: str, status: str) -> list:
    client = MongoClient(MONGO_URI)
    db = client[db_name]
    orders = list(db.orders.find(
        {"status": status},
        {"_id": 0, "orderId": 1, "amount": 1, "customerId": 1}
    ))
    client.close()
    return orders
```

The `retries=3` decorator automatically retries on transient MongoDB connection errors.

## Transforming Data in a Task

```python
@task
def calculate_totals(orders: list) -> list:
    return [
        {**order, "totalWithTax": round(order["amount"] * 1.08, 2)}
        for order in orders
    ]
```

## Writing Results to MongoDB

```python
@task(retries=2)
def write_results(db_name: str, collection: str, records: list) -> int:
    if not records:
        return 0
    client = MongoClient(MONGO_URI)
    result = client[db_name][collection].insert_many(records)
    client.close()
    return len(result.inserted_ids)
```

## Building a Full Flow

```python
@flow(name="Daily Order Processing", log_prints=True)
def process_orders(db_name: str = "myDatabase"):
    orders = read_orders(db_name=db_name, status="pending")
    print(f"Read {len(orders)} orders")

    if not orders:
        print("No orders to process")
        return

    enriched = calculate_totals(orders=orders)
    count = write_results(db_name=db_name, collection="processed_orders", records=enriched)
    print(f"Wrote {count} records")

if __name__ == "__main__":
    process_orders()
```

## Scheduling the Flow

Deploy the flow with a schedule using Prefect's deployment system:

```python
from prefect.deployments import Deployment
from prefect.server.schemas.schedules import CronSchedule

deployment = Deployment.build_from_flow(
    flow=process_orders,
    name="daily-order-run",
    schedule=CronSchedule(cron="0 2 * * *", timezone="UTC"),
    parameters={"db_name": "myDatabase"},
)

deployment.apply()
```

This schedules the flow to run at 2 AM UTC every day.

## Using Prefect Blocks for MongoDB Credentials

Store connection strings securely as Prefect Blocks instead of hardcoding them:

```python
from prefect.blocks.system import Secret

# Register the secret (done once)
Secret(value="mongodb+srv://user:pass@cluster.mongodb.net/").save("mongo-uri")
```

```python
@task
def get_mongo_uri() -> str:
    return Secret.load("mongo-uri").get()
```

## Running with the Prefect UI

Start the Prefect server:

```bash
prefect server start
```

Deploy and run:

```bash
prefect deployment run "Daily Order Processing/daily-order-run"
```

The Prefect UI at `http://localhost:4200` shows flow run status, logs per task, and retry history.

## Handling Upserts

For idempotent pipelines, use upsert instead of insert_many:

```python
@task(retries=2)
def upsert_results(db_name: str, records: list) -> None:
    client = MongoClient(MONGO_URI)
    collection = client[db_name].processed_orders
    for record in records:
        collection.update_one(
            {"orderId": record["orderId"]},
            {"$set": record},
            upsert=True
        )
    client.close()
```

## Summary

Prefect wraps MongoDB operations in tasks with built-in retry logic, logging, and scheduling. Define tasks for reading, transforming, and writing MongoDB data, compose them in a flow, and deploy with a schedule. Use Prefect Blocks to manage MongoDB connection strings securely and the Prefect UI to monitor run history and diagnose failures.
