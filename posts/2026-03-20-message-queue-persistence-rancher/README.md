# How to Configure Message Queue Persistence in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Message Persistence, RabbitMQ, Kafka, Kubernetes, Storage

Description: Configure durable message persistence for RabbitMQ and Kafka in Rancher using persistent volumes, durable queues, and replication policies.

## Introduction

Message persistence ensures messages survive broker restarts and pod failures. Without persistence, a broker restart causes message loss. This guide covers persistence configuration for both RabbitMQ and Kafka on Rancher.

## RabbitMQ Persistence

RabbitMQ persistence operates at two levels: broker-level storage via PVCs, and queue-level durability via the `durable` flag.

### Broker Storage Configuration

```yaml
# rabbitmq-values.yaml (persistence section)

persistence:
  enabled: true                # Enable PVC creation
  storageClass: "longhorn"
  size: 50Gi
  accessMode: ReadWriteOnce

# Important: Set the message store paths
extraEnvVars:
  - name: RABBITMQ_MNESIA_DIR
    value: /bitnami/rabbitmq/mnesia   # Maps to the PVC mount
```

### Declare Durable Queues

Queues must be declared as durable, and messages must be marked persistent:

```python
# Python example using pika library
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('rabbitmq.messaging.svc.cluster.local')
)
channel = connection.channel()

# Declare a durable queue
channel.queue_declare(
    queue='orders',
    durable=True    # Queue survives broker restart
)

# Publish a persistent message
channel.basic_publish(
    exchange='',
    routing_key='orders',
    body='Order #12345',
    properties=pika.BasicProperties(
        delivery_mode=pika.spec.PERSISTENT_DELIVERY_MODE   # Message survives restart
    )
)
```

### RabbitMQ Quorum Queues (Recommended for Production)

Quorum queues replace classic mirrored queues and provide stronger consistency guarantees:

```python
# Declare a quorum queue
channel.queue_declare(
    queue='orders-quorum',
    durable=True,
    arguments={'x-queue-type': 'quorum'}   # Use Raft-based replication
)
```

## Kafka Persistence

Kafka persists messages to disk automatically. Key settings control retention.

### Broker Storage Configuration

```yaml
# kafka-values.yaml (persistence section)
persistence:
  enabled: true
  storageClass: "longhorn"
  size: 100Gi   # Kafka is very storage-intensive

# Kafka log configuration
config: |
  log.retention.hours=168        # Keep messages for 7 days
  log.retention.bytes=10737418240  # Keep up to 10GB per partition
  log.segment.bytes=1073741824   # Roll segment files at 1GB
  log.cleanup.policy=delete      # Delete old segments (vs compact)
```

### Topic-Level Retention Override

```bash
# Set different retention for a specific topic
kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name audit-log \
  --alter \
  --add-config "retention.ms=2592000000"  # 30 days for audit logs
```

### Replication Factor

For production topics, always set replication factor to 3:

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic payments \
  --partitions 6 \
  --replication-factor 3 \
  --config min.insync.replicas=2
```

## Conclusion

Message persistence in Rancher requires correctly configuring both the storage layer (PVCs with `Retain` policy) and the broker-level settings (durable queues, message persistence flags, replication). The combination ensures zero message loss even during pod restarts or node failures.
