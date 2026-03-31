# How to Stream MongoDB Changes to Amazon Kinesis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Kinesis, AWS, Change Stream, Streaming

Description: Learn how to stream MongoDB change events to Amazon Kinesis Data Streams using change streams and AWS Lambda or a custom producer for real-time pipelines.

---

## Overview

Streaming MongoDB change events to Amazon Kinesis enables real-time AWS-native pipelines - feeding analytics, triggering Lambda functions, or loading data into Redshift or S3 via Kinesis Data Firehose. The integration is achieved by consuming MongoDB change streams and publishing records to Kinesis using the AWS SDK.

## Prerequisites

- MongoDB 4.0+ replica set (change streams require a replica set)
- AWS account with Kinesis Data Streams configured
- IAM role or user with `kinesis:PutRecord` and `kinesis:PutRecords` permissions

```bash
# Create Kinesis Data Stream via AWS CLI
aws kinesis create-stream \
  --stream-name mongodb-changes \
  --shard-count 2 \
  --region us-east-1
```

## Python Producer: Change Stream to Kinesis

```python
import boto3, json, time
from pymongo import MongoClient
from bson import json_util
from datetime import datetime, timezone

mongo = MongoClient("mongodb://localhost:27017/?replicaSet=rs0")
db = mongo["mydb"]
kinesis = boto3.client("kinesis", region_name="us-east-1")
STREAM_NAME = "mongodb-changes"

def serialize_event(change_event):
  # Use bson.json_util to handle ObjectId and ISODate serialization
  return json_util.dumps({
    "operationType": change_event["operationType"],
    "fullDocument": change_event.get("fullDocument"),
    "documentKey": change_event.get("documentKey"),
    "updateDescription": change_event.get("updateDescription"),
    "occurredAt": datetime.now(timezone.utc).isoformat()
  })

def stream_to_kinesis(collection_name):
  collection = db[collection_name]
  pipeline = [{ "$match": { "operationType": { "$in": ["insert", "update", "replace", "delete"] } } }]

  with collection.watch(pipeline, full_document="updateLookup") as stream:
    print(f"Listening for changes on {collection_name}...")
    records = []
    last_flush = time.time()

    for change in stream:
      aggregate_id = str(change["documentKey"]["_id"])
      records.append({
        "Data": serialize_event(change).encode("utf-8"),
        "PartitionKey": aggregate_id  # route related events to same shard
      })

      # Flush in batches of 500 or every 1 second
      if len(records) >= 500 or (time.time() - last_flush) >= 1.0:
        response = kinesis.put_records(StreamName=STREAM_NAME, Records=records)
        failed = response.get("FailedRecordCount", 0)
        print(f"Flushed {len(records)} records, {failed} failed")
        records = []
        last_flush = time.time()

stream_to_kinesis("orders")
```

## Resume Tokens for Fault Tolerance

Store the change stream resume token in MongoDB to survive restarts:

```python
def get_resume_token(db):
  doc = db.metadata.find_one({"_id": "orders_kinesis_resume"})
  return doc["resumeToken"] if doc else None

def save_resume_token(db, token):
  db.metadata.replace_one(
    {"_id": "orders_kinesis_resume"},
    {"_id": "orders_kinesis_resume", "resumeToken": token},
    upsert=True
  )

# Resume from saved token
resume_token = get_resume_token(db)
watch_options = {"resume_after": resume_token} if resume_token else {}

with db.orders.watch(pipeline, full_document="updateLookup", **watch_options) as stream:
  for change in stream:
    # ... publish to Kinesis ...
    save_resume_token(db, change["_id"])  # save after successful publish
```

## Consume Kinesis Events with Lambda

Attach a Lambda function to the Kinesis stream to process MongoDB changes in real time:

```python
# Lambda function handler
import json
import base64
from bson import json_util

def handler(event, context):
  for record in event["Records"]:
    payload = base64.b64decode(record["kinesis"]["data"]).decode("utf-8")
    change = json_util.loads(payload)

    op = change["operationType"]
    doc = change.get("fullDocument", {})
    print(f"Processing: {op} for document {change['documentKey']['_id']}")

    if op == "insert":
      process_new_order(doc)
    elif op in ("update", "replace"):
      process_updated_order(doc)
    elif op == "delete":
      process_deleted_order(change["documentKey"]["_id"])
```

## Monitor the Pipeline

```bash
# Check Kinesis stream metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kinesis \
  --metric-name PutRecords.Records \
  --dimensions Name=StreamName,Value=mongodb-changes \
  --start-time 2026-03-31T00:00:00Z \
  --end-time 2026-03-31T23:59:59Z \
  --period 300 \
  --statistics Sum \
  --region us-east-1
```

## Summary

Streaming MongoDB changes to Kinesis combines MongoDB change streams with the Kinesis SDK to build AWS-native real-time pipelines. Use `PartitionKey` based on document ID for ordering guarantees within an aggregate, store resume tokens in MongoDB for fault-tolerant restarts, and batch records to minimize API calls. Downstream Kinesis consumers (Lambda, Firehose, KCL) can then process events in real time.
