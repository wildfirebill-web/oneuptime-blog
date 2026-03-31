# How to Connect Atlas Stream Processing to Kafka

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Kafka, Stream Processing, Integration

Description: Learn how to connect MongoDB Atlas Stream Processing to Apache Kafka as a source or sink to build real-time data pipelines between Kafka topics and MongoDB collections.

---

## Why Connect Atlas Stream Processing to Kafka

Apache Kafka is a widely used distributed event streaming platform. Connecting MongoDB Atlas Stream Processing to Kafka allows you to consume events from Kafka topics, apply real-time transformations and aggregations using MongoDB's aggregation pipeline syntax, and write results to MongoDB Atlas collections - all without managing separate stream processing infrastructure.

## Prerequisites

- An Atlas Stream Processing instance (M10 or higher)
- A running Kafka cluster (self-hosted, Confluent Cloud, or MSK)
- Network connectivity between Atlas and your Kafka brokers

## Creating a Kafka Connection in Atlas

In the Atlas UI, navigate to your stream processing instance, then go to Connections and add a new Kafka connection.

For self-hosted Kafka, configure the connection via the Atlas CLI:

```bash
# Create a Kafka connection for Atlas Stream Processing
atlas streams connections create \
  --instance myStreamInstance \
  --type Kafka \
  --name myKafkaSource \
  --bootstrapServers "broker1:9092,broker2:9092" \
  --security.protocol SASL_SSL \
  --sasl.mechanism PLAIN \
  --sasl.username "$KAFKA_USERNAME" \
  --sasl.password "$KAFKA_PASSWORD"
```

## Reading from a Kafka Topic as $source

Use the `$source` stage to read messages from a Kafka topic. Atlas Stream Processing automatically deserializes JSON payloads.

```javascript
[
  {
    $source: {
      connectionName: "myKafkaSource",
      topic: "order-events",
      timeField: {
        $dateFromString: {
          dateString: "$$ROOT.eventTimestamp"
        }
      }
    }
  },
  {
    $match: {
      eventType: "purchase"
    }
  },
  {
    $project: {
      orderId: 1,
      customerId: 1,
      amount: 1,
      eventTimestamp: 1
    }
  },
  {
    $merge: {
      into: {
        connectionName: "atlasCluster",
        db: "orders",
        coll: "purchases"
      }
    }
  }
]
```

## Writing to a Kafka Topic as $emit

Use `$emit` to write processed stream documents back to a Kafka topic:

```javascript
[
  {
    $source: {
      connectionName: "myKafkaSource",
      topic: "raw-sensor-data"
    }
  },
  {
    $tumblingWindow: {
      interval: { size: 1, unit: "minute" },
      pipeline: [
        {
          $group: {
            _id: "$deviceId",
            avgTemp: { $avg: "$temperature" },
            alertCount: {
              $sum: {
                $cond: [{ $gt: ["$temperature", 80] }, 1, 0]
              }
            }
          }
        }
      ]
    }
  },
  {
    $emit: {
      connectionName: "myKafkaSource",
      topic: "aggregated-sensor-metrics"   // Write results to a different topic
    }
  }
]
```

## Handling Kafka Message Schema

Atlas Stream Processing expects messages to be JSON. For Avro or Protobuf schemas, configure a schema registry connection or decode the payload in the pipeline using `$function`.

```javascript
// If your Kafka messages use a timestamp field with epoch milliseconds
{
  $source: {
    connectionName: "myKafkaSource",
    topic: "events",
    timeField: {
      $toDate: "$$ROOT.timestampMs"   // Convert epoch ms to Date
    }
  }
}
```

## Monitoring the Pipeline

Check pipeline status and throughput via the Atlas UI or CLI:

```bash
# List active stream processing pipelines
atlas streams pipelines list --instance myStreamInstance

# View pipeline statistics
atlas streams pipelines describe myPipelineName --instance myStreamInstance
```

## Summary

Connecting MongoDB Atlas Stream Processing to Apache Kafka enables you to build real-time event processing pipelines using familiar MongoDB aggregation syntax. Use `$source` with a Kafka connection to consume topic messages, apply filtering, transformation, and windowed aggregations, and write results to Atlas collections with `$merge` or back to Kafka with `$emit`. Configure SASL authentication and ensure broker connectivity before creating your connection, and monitor pipeline throughput through the Atlas UI.
