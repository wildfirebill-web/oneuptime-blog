# How to Set Up MongoDB as a Source in Apache NiFi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, NiFi, ETL, Data Pipeline, Integration

Description: Learn how to configure Apache NiFi to read documents from MongoDB as a source for data pipeline flows using the GetMongo processor.

---

## Why Use MongoDB as a NiFi Source?

Apache NiFi is a data flow platform that moves, transforms, and routes data between systems. Using MongoDB as a source lets you stream documents into NiFi for processing, transformation, and delivery to destinations like Kafka, S3, HDFS, or relational databases.

## Prerequisites

- Apache NiFi 1.23+ or 2.x
- MongoDB 5.0+ with network access from NiFi
- The `nifi-mongodb-bundle` NAR file (included in standard NiFi distributions)

## Adding the MongoDB Controller Service

NiFi uses a controller service to manage the MongoDB connection:

1. In the NiFi UI, open the Controller Services panel (gear icon)
2. Add a new MongoDBControllerService
3. Configure it:

```text
URI:       mongodb+srv://user:pass@cluster.mongodb.net/
Database:  myDatabase
```

Enable the service after saving.

## GetMongo Processor - Querying Documents

The `GetMongo` processor executes a query and emits each matching document as a FlowFile:

1. Add a GetMongo processor to the canvas
2. Set the properties:

```text
MongoDB Service: (select your controller service)
Database Name:   myDatabase
Collection Name: orders
Query:           { "status": "pending" }
Projection:      { "_id": 1, "orderId": 1, "amount": 1 }
Sort:            { "createdAt": 1 }
Limit:           1000
```

The processor emits one FlowFile per matching document containing the document as JSON.

## Batch Mode

For large collections, enable the Batch Size property to control how many documents are fetched per poll cycle:

```text
Batch Size: 500
```

## Incremental Reads with State

To read new documents continuously without re-reading old ones, use a timestamp or ObjectId cursor field in the query and store state using NiFi's State Management. A common pattern:

```json
{ "createdAt": { "$gt": { "$date": "${last_read_date}" } } }
```

Use the `UpdateAttribute` processor to track the latest timestamp after each successful read.

## GetMongoRecord Processor

For structured processing with NiFi's Record API (which enables schema-aware transformations), use `GetMongoRecord` instead of `GetMongo`:

```text
MongoDB Service:   (controller service)
Database Name:     myDatabase
Collection Name:   products
Record Reader:     JsonTreeReader
Record Writer:     AvroRecordSetWriter
Schema Name:       product_schema
```

This outputs records in Avro or Parquet format, suitable for writing directly to HDFS or S3.

## A Simple NiFi Flow: MongoDB to Kafka

```text
[GetMongo] --> [SplitJson] --> [PublishKafka]
```

- GetMongo reads orders from MongoDB
- SplitJson splits a batch FlowFile into individual documents
- PublishKafka sends each order to a Kafka topic

For GetMongo, set the output to return an array by toggling the JSON Type property.

## Handling Errors

Connect the failure relationship of GetMongo to a PutFile or LogAttribute processor to capture and inspect failed queries:

```text
[GetMongo] --failure--> [LogAttribute]
             --success--> [PublishKafka]
```

## Monitoring the Flow

Use NiFi's bulletin board and data provenance to track:
- Number of FlowFiles queued
- Processing throughput (documents per second)
- Error rates on the failure relationship

## Summary

Setting up MongoDB as a NiFi source requires a MongoDBControllerService and a GetMongo or GetMongoRecord processor. Use query and projection parameters to limit data at the source, batch size to control throughput, and state management for incremental reads. Connect the success relationship to downstream processors for transformation and delivery.
