# How to Set Up Bucket Notifications in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Notification, Event, Object Storage, Kafka, AMQP

Description: Configure Ceph RGW bucket notifications to publish object events to HTTP endpoints, Kafka, or AMQP brokers for real-time event processing.

---

Ceph RGW bucket notifications allow you to publish events triggered by object operations (PUT, DELETE, COPY, etc.) to external endpoints. This is compatible with the AWS S3 notification API and supports HTTP/HTTPS, Kafka, and AMQP as targets.

## Supported Notification Targets

- **HTTP/HTTPS**: POST events as JSON to any web endpoint
- **Kafka**: Publish events to Kafka topics
- **AMQP**: Publish events to RabbitMQ or other AMQP brokers

## Creating an SNS Topic (HTTP Endpoint)

Bucket notifications in RGW use an SNS-like topic abstraction. First create a topic pointing to your endpoint:

```bash
aws sns create-topic \
  --name my-http-topic \
  --attributes '{"push-endpoint": "http://webhook.example.com/notify"}' \
  --endpoint-url http://your-rgw-host:7480
```

For Kafka:

```bash
aws sns create-topic \
  --name my-kafka-topic \
  --attributes '{"push-endpoint": "kafka://kafka.example.com:9092", "kafka-ack-level": "broker"}' \
  --endpoint-url http://your-rgw-host:7480
```

## Subscribing a Bucket to a Topic

After creating a topic, configure the bucket notification:

```bash
aws s3api put-bucket-notification-configuration \
  --bucket mybucket \
  --notification-configuration '{
    "TopicConfigurations": [
      {
        "Id": "object-created",
        "TopicArn": "arn:aws:sns:default::my-http-topic",
        "Events": ["s3:ObjectCreated:*", "s3:ObjectRemoved:*"]
      }
    ]
  }' \
  --endpoint-url http://your-rgw-host:7480
```

## Filtering Events by Object Key

Narrow notifications to specific key prefixes or suffixes:

```bash
aws s3api put-bucket-notification-configuration \
  --bucket mybucket \
  --notification-configuration '{
    "TopicConfigurations": [
      {
        "Id": "images-only",
        "TopicArn": "arn:aws:sns:default::my-kafka-topic",
        "Events": ["s3:ObjectCreated:Put"],
        "Filter": {
          "Key": {
            "FilterRules": [
              {"Name": "prefix", "Value": "images/"},
              {"Name": "suffix", "Value": ".jpg"}
            ]
          }
        }
      }
    ]
  }' \
  --endpoint-url http://your-rgw-host:7480
```

## Listing and Deleting Topics

```bash
# List topics
aws sns list-topics --endpoint-url http://your-rgw-host:7480

# Delete a topic
aws sns delete-topic \
  --topic-arn arn:aws:sns:default::my-http-topic \
  --endpoint-url http://your-rgw-host:7480
```

## Sample Event Payload

When an object is created, RGW sends a JSON payload similar to:

```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "ceph:s3",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {"name": "mybucket"},
        "object": {"key": "images/photo.jpg", "size": 204800}
      }
    }
  ]
}
```

## Summary

Ceph RGW bucket notifications provide an S3-compatible event streaming API supporting HTTP, Kafka, and AMQP targets. Create SNS topics pointing to your endpoint, then configure bucket notification rules with optional key filters. This enables real-time pipelines that react to object storage events without polling.
