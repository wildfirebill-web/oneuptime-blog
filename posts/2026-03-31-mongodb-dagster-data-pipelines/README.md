# How to Use MongoDB with Dagster for Data Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Dagster, Data Pipeline, Asset, Python

Description: Learn how to build Dagster software-defined assets and ops that read from and write to MongoDB for observable, testable data pipelines.

---

## Why Dagster with MongoDB?

Dagster is a data orchestration platform built around the concept of software-defined assets (SDAs). Unlike task-based orchestrators, Dagster models data assets as first-class citizens, making lineage, testing, and observability natural. MongoDB integrates cleanly as an I/O Manager or via direct PyMongo calls in assets.

## Installing Dependencies

```bash
pip install dagster dagster-webserver pymongo
```

## Defining a MongoDB Resource

```python
from dagster import resource, InitResourceContext
from pymongo import MongoClient

@resource(config_schema={"uri": str, "database": str})
def mongo_resource(context: InitResourceContext):
    client = MongoClient(context.resource_config["uri"])
    db = client[context.resource_config["database"]]
    try:
        yield db
    finally:
        client.close()
```

## Building a Software-Defined Asset

```python
from dagster import asset, AssetExecutionContext

@asset(required_resource_keys={"mongo"})
def raw_orders(context: AssetExecutionContext) -> list:
    db = context.resources.mongo
    orders = list(db.orders.find(
        {"status": "completed"},
        {"_id": 0, "orderId": 1, "amount": 1, "customerId": 1}
    ))
    context.log.info(f"Loaded {len(orders)} orders from MongoDB")
    return orders
```

## A Downstream Asset That Writes to MongoDB

```python
@asset(required_resource_keys={"mongo"})
def enriched_orders(context: AssetExecutionContext, raw_orders: list) -> None:
    db = context.resources.mongo
    enriched = [
        {**o, "totalWithTax": round(o["amount"] * 1.08, 2), "processed": True}
        for o in raw_orders
    ]
    if enriched:
        db.processed_orders.insert_many(enriched)
    context.log.info(f"Wrote {len(enriched)} enriched orders")
```

## Wiring Assets with Definitions

```python
from dagster import Definitions

defs = Definitions(
    assets=[raw_orders, enriched_orders],
    resources={
        "mongo": mongo_resource.configured({
            "uri": "mongodb+srv://user:pass@cluster.mongodb.net/",
            "database": "myDatabase"
        })
    }
)
```

## Running the Pipeline

Start the Dagster UI:

```bash
dagster dev -m pipeline_module
```

Navigate to `http://localhost:3000` to see the asset graph and trigger materializations.

## Ops-Based Approach

For non-asset workflows, use ops and jobs:

```python
from dagster import op, job, Out

@op(required_resource_keys={"mongo"}, out=Out(list))
def fetch_users(context) -> list:
    return list(context.resources.mongo.users.find(
        {"active": True}, {"_id": 0, "email": 1, "name": 1}
    ))

@op(required_resource_keys={"mongo"})
def write_report(context, users: list) -> None:
    context.resources.mongo.reports.insert_one({
        "date": "2026-03-31",
        "activeUserCount": len(users),
        "users": users
    })

@job(resource_defs={"mongo": mongo_resource})
def user_report_job():
    write_report(fetch_users())
```

## Scheduling with Dagster

```python
from dagster import ScheduleDefinition

daily_schedule = ScheduleDefinition(
    job=user_report_job,
    cron_schedule="0 6 * * *",
)

defs = Definitions(
    jobs=[user_report_job],
    schedules=[daily_schedule],
    resources={"mongo": mongo_resource.configured({
        "uri": "mongodb://localhost:27017",
        "database": "myDatabase"
    })}
)
```

## Testing Assets

Dagster makes unit testing straightforward by allowing resource injection:

```python
from dagster import build_asset_context

def test_enriched_orders():
    mock_orders = [{"orderId": "001", "amount": 100.0, "customerId": "c1"}]

    class MockCollection:
        def insert_many(self, docs):
            assert len(docs) == 1
            assert docs[0]["totalWithTax"] == 108.0

    class MockDB:
        processed_orders = MockCollection()

    ctx = build_asset_context(resources={"mongo": MockDB()})
    enriched_orders(ctx, raw_orders=mock_orders)
```

## Summary

Dagster's resource system makes MongoDB integration clean and testable. Define a MongoDB resource, use it in software-defined assets for lineage-aware pipelines, and compose assets into jobs with schedules. The Dagster UI provides a visual asset graph, materialization history, and logs per asset run - making MongoDB-backed pipelines easy to observe and debug.
