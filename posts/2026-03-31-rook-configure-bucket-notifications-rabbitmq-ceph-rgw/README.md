# How to Configure Bucket Notifications to RabbitMQ in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, RGW, RabbitMQ, Notification, Event, Object Storage

Description: Configure Ceph RGW bucket notifications to publish S3 object events to a RabbitMQ AMQP exchange for event-driven workflows.

---

## Overview

Ceph RGW can publish bucket events (object created, removed, etc.) to RabbitMQ via AMQP. This is useful when you need reliable message delivery with routing flexibility using RabbitMQ exchanges and bindings.

## Prerequisites

- Ceph RGW (Nautilus or later) with SNS notification support
- RabbitMQ server accessible from RGW hosts
- RabbitMQ exchange already created (or let RGW create it)
- AWS CLI configured for RGW

## Step 1: Create a RabbitMQ Exchange in RabbitMQ

Use the RabbitMQ management API or CLI to create an exchange:

```bash
rabbitmqadmin declare exchange \
  name=s3-events \
  type=topic \
  durable=true \
  --host=rabbitmq.example.com \
  --username=admin \
  --password=secret
```

## Step 2: Create an SNS Topic Pointing to RabbitMQ

RGW uses an SNS-compatible API. Create a topic that targets your RabbitMQ AMQP endpoint:

```bash
aws sns create-topic \
  --name s3-events \
  --attributes '{
    "push-endpoint": "amqp://user:password@rabbitmq.example.com:5672",
    "amqp-exchange": "s3-events",
    "amqp-ack-level": "broker"
  }' \
  --endpoint-url http://rgw.example.com:7480
```

Note the returned topic ARN for use in the next step.

## Step 3: Configure Bucket Notification

Create the notification configuration JSON:

```json
{
  "TopicConfigurations": [
    {
      "Id": "rabbitmq-all-events",
      "TopicArn": "arn:aws:sns:default::s3-events",
      "Events": ["s3:ObjectCreated:*", "s3:ObjectRemoved:*"]
    }
  ]
}
```

Apply to the bucket:

```bash
aws s3api put-bucket-notification-configuration \
  --bucket my-events-bucket \
  --notification-configuration file://notification.json \
  --endpoint-url http://rgw.example.com:7480
```

## Step 4: Bind a Queue and Test

In RabbitMQ, create a queue and bind it to the exchange:

```bash
rabbitmqadmin declare queue name=s3-queue durable=true \
  --host=rabbitmq.example.com --username=admin --password=secret

rabbitmqadmin declare binding \
  source=s3-events \
  destination=s3-queue \
  routing_key=# \
  --host=rabbitmq.example.com --username=admin --password=secret
```

Upload a test object:

```bash
aws s3 cp test.txt s3://my-events-bucket/ \
  --endpoint-url http://rgw.example.com:7480
```

Check for the message in RabbitMQ:

```bash
rabbitmqadmin get queue=s3-queue \
  --host=rabbitmq.example.com --username=admin --password=secret
```

## Using SSL for Secure AMQP

For production, use AMQPS with SSL:

```bash
aws sns create-topic \
  --name s3-events-secure \
  --attributes '{
    "push-endpoint": "amqps://user:password@rabbitmq.example.com:5671",
    "amqp-exchange": "s3-events",
    "amqp-ack-level": "broker",
    "verify-ssl": "true",
    "ca-location": "/etc/ceph/rabbitmq-ca.crt"
  }' \
  --endpoint-url http://rgw.example.com:7480
```

## Summary

Ceph RGW sends bucket event notifications to RabbitMQ by creating an SNS topic with an AMQP push endpoint, then attaching the topic to a bucket via `put-bucket-notification-configuration`. Events flow through the RabbitMQ exchange to bound queues, enabling consumers to process S3 events with full AMQP routing flexibility.
