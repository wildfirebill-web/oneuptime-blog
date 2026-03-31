# How to Set Up Bucket Notifications in Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, Bucket Notifications, Kafka, S3, Kubernetes

Description: Learn how to configure bucket notifications in Rook Ceph Object Store to send events to Kafka, HTTP endpoints, or AMQP when objects are created or deleted.

---

## Overview

Bucket notifications allow the Ceph RGW to emit events to external systems when objects are created, deleted, or modified. Rook exposes this capability through the `CephBucketNotification` and `CephBucketTopic` CRDs, which map to S3-compatible notification configurations.

Supported notification targets include:
- Kafka (recommended for production)
- HTTP/HTTPS endpoints
- AMQP (RabbitMQ)

## Step 1 - Create a CephBucketTopic

Define the target endpoint where events will be sent:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBucketTopic
metadata:
  name: my-topic
  namespace: rook-ceph
spec:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
  endpoint:
    kafka:
      uri: kafka://kafka-broker.kafka.svc:9092
      useSSL: false
      disableVerifySSL: false
      ackLevel: broker
```

Apply the topic:

```bash
kubectl apply -f bucket-topic.yaml
```

Verify the topic was created:

```bash
kubectl -n rook-ceph get cephbuckettopic my-topic
```

## Step 2 - Create a CephBucketNotification

Define which bucket events trigger a notification and link to the topic:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBucketNotification
metadata:
  name: my-notification
  namespace: rook-ceph
spec:
  topic: my-topic
  events:
    - s3:ObjectCreated:*
    - s3:ObjectRemoved:*
  filter:
    keyFilter:
      filterRules:
        - name: prefix
          value: images/
        - name: suffix
          value: .jpg
```

Apply the notification:

```bash
kubectl apply -f bucket-notification.yaml
```

## Step 3 - Associate Notification with a Bucket

Link the notification to an ObjectBucketClaim:

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: my-bucket
  namespace: default
  labels:
    bucket-notification-my-notification: my-notification
spec:
  generateBucketName: notify-bucket
  storageClassName: rook-ceph-bucket
```

Alternatively, add the label to an existing OBC:

```bash
kubectl label obc my-bucket bucket-notification-my-notification=my-notification
```

## Step 4 - Verify Notification Configuration

Check the notification is configured on the bucket via the radosgw-admin CLI:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin notification list --bucket=notify-bucket-abc123
```

Expected output:

```text
[
    {
        "Id": "my-notification",
        "TopicArn": "arn:aws:sns:default::my-topic",
        "Events": ["s3:ObjectCreated:*", "s3:ObjectRemoved:*"],
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
```

## Testing Notifications

Upload a file to trigger a notification:

```bash
aws --endpoint-url http://<rgw-ip>:80 \
  s3 cp test.jpg s3://notify-bucket-abc123/images/test.jpg
```

Then verify the Kafka topic received the event:

```bash
kafka-console-consumer.sh \
  --bootstrap-server kafka-broker.kafka.svc:9092 \
  --topic my-topic \
  --from-beginning \
  --max-messages 1
```

## Summary

Setting up bucket notifications in Rook requires creating a CephBucketTopic (defining the target endpoint), a CephBucketNotification (defining event types and filters), and associating the notification with a bucket via OBC labels. Events are emitted in S3-compatible notification format, making them consumable by standard message processing pipelines connected to Kafka, HTTP, or AMQP endpoints.
