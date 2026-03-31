# How to Use MongoDB with Luigi for Data Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Luigi, Data Pipeline, ETL, Python

Description: Learn how to build Luigi data pipeline tasks that read from and write to MongoDB collections for batch data processing workflows.

---

## What Is Luigi?

Luigi is a Python framework for building data pipelines as dependency graphs. Each unit of work is a `Task` class with `requires()`, `output()`, and `run()` methods. Luigi handles task scheduling, dependency resolution, and failure recovery. Integrating MongoDB lets you use it as a task input or output store.

## Installing Dependencies

```bash
pip install luigi pymongo
```

## A Simple MongoDB Read Task

```python
import luigi
import pymongo
import json

MONGO_URI = "mongodb://localhost:27017/"

class ExtractOrders(luigi.Task):
    date = luigi.DateParameter()

    def output(self):
        return luigi.LocalTarget(f"/tmp/orders_{self.date}.json")

    def run(self):
        client = pymongo.MongoClient(MONGO_URI)
        db = client["myDatabase"]
        orders = list(db.orders.find(
            {"orderDate": {"$gte": str(self.date)}},
            {"_id": 0, "orderId": 1, "amount": 1, "status": 1}
        ))
        client.close()

        with self.output().open("w") as f:
            json.dump(orders, f)
```

## A MongoDB Write Task

```python
class LoadProcessedOrders(luigi.Task):
    date = luigi.DateParameter()

    def requires(self):
        return TransformOrders(date=self.date)

    def output(self):
        return luigi.LocalTarget(f"/tmp/loaded_{self.date}.done")

    def run(self):
        with self.input().open("r") as f:
            records = json.load(f)

        client = pymongo.MongoClient(MONGO_URI)
        db = client["myDatabase"]
        if records:
            db.processed_orders.insert_many(records)
        client.close()

        with self.output().open("w") as f:
            f.write("done")
```

## A Transformation Task

```python
class TransformOrders(luigi.Task):
    date = luigi.DateParameter()

    def requires(self):
        return ExtractOrders(date=self.date)

    def output(self):
        return luigi.LocalTarget(f"/tmp/transformed_{self.date}.json")

    def run(self):
        with self.input().open("r") as f:
            orders = json.load(f)

        transformed = [
            {
                **order,
                "totalWithTax": round(order["amount"] * 1.08, 2),
                "processed": True,
            }
            for order in orders
        ]

        with self.output().open("w") as f:
            json.dump(transformed, f)
```

## A Full ETL Pipeline

```python
class DailyOrderPipeline(luigi.WrapperTask):
    date = luigi.DateParameter()

    def requires(self):
        return LoadProcessedOrders(date=self.date)
```

Run the pipeline from the command line:

```bash
python -m luigi --module pipeline DailyOrderPipeline \
  --date 2026-03-31 \
  --local-scheduler
```

## Using a MongoDB Target

For more robust task completion tracking, implement a custom Luigi Target that checks MongoDB directly:

```python
class MongoTarget(luigi.Target):
    def __init__(self, uri, db, collection, query):
        self.uri = uri
        self.db = db
        self.collection = collection
        self.query = query

    def exists(self):
        client = pymongo.MongoClient(self.uri)
        count = client[self.db][self.collection].count_documents(self.query)
        client.close()
        return count > 0
```

```python
class LoadOrders(luigi.Task):
    date = luigi.DateParameter()

    def output(self):
        return MongoTarget(
            uri=MONGO_URI,
            db="myDatabase",
            collection="pipeline_runs",
            query={"date": str(self.date), "task": "LoadOrders"}
        )
```

## Running Luigi with the Central Scheduler

For production pipelines, start the Luigi scheduler:

```bash
luigid --background --logdir /var/log/luigi
```

Then submit tasks:

```bash
python -m luigi --module pipeline DailyOrderPipeline \
  --date 2026-03-31 \
  --scheduler-host localhost
```

## Summary

Luigi's task-based model integrates cleanly with MongoDB through PyMongo. Define extraction tasks that read from MongoDB to local files, transformation tasks for processing, and load tasks that write back to MongoDB. Use a custom MongoTarget for idempotent pipelines that check collection state rather than file existence to determine task completion.
