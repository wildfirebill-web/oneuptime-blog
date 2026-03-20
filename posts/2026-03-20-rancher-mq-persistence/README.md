# How to Configure Message Queue Persistence in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Message Queue, Persistence, Storage, Durability

Description: Configure persistent message storage for RabbitMQ, Kafka, and NATS on Rancher to ensure messages survive pod restarts and node failures.

## Introduction

Message persistence ensures that messages are not lost when broker pods restart, nodes fail, or maintenance is performed. Each message queue system handles persistence differently. This guide covers configuring durable storage for RabbitMQ, Kafka, and NATS on Rancher-managed clusters.

## Prerequisites

- Rancher-managed cluster
- A StorageClass with persistent volumes
- kubectl access

## Section 1: RabbitMQ Persistence

### Configure Durable Queues and Messages

```yaml
# rabbitmq-persistence-values.yaml - Persistent RabbitMQ configuration
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-persistent
  namespace: messaging
spec:
  replicas: 3
  persistence:
    # Critical: enable persistent storage
    storageClassName: standard
    storage: 20Gi
  rabbitmq:
    additionalConfig: |
      # Use durable queues by default
      queue.default_queue_type = quorum

      # AOF-style persistence for message journals
      msg_store_index_module = rabbit_msg_store_ets_index

      # Flush journal to disk
      queue_journal_max_bytes_size = 0

      # Checkpoint interval (ms)
      queue_persistent_file_delay = 100
```

### Publish Persistent Messages

```python
# publisher.py - Example of publishing persistent messages
import pika
import json

connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='rabbitmq-persistent.messaging.svc.cluster.local',
        credentials=pika.PlainCredentials('appuser', 'AppUserP@ss')
    )
)
channel = connection.channel()

# Declare a durable queue
channel.queue_declare(
    queue='orders',
    durable=True,  # Queue survives broker restart
    arguments={
        'x-queue-type': 'quorum'  # Quorum queues are always durable
    }
)

# Publish with persistent delivery mode
channel.basic_publish(
    exchange='',
    routing_key='orders',
    body=json.dumps({'order_id': '123', 'item': 'widget'}),
    properties=pika.BasicProperties(
        delivery_mode=pika.DeliveryMode.Persistent,  # Message survives restart
        content_type='application/json'
    )
)
```

## Section 2: Kafka Persistence

### Configure Kafka Log Retention

```yaml
# kafka-persistence.yaml - Kafka with persistent storage configuration
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka-persistent
  namespace: kafka
spec:
  kafka:
    config:
      # Retain logs for 7 days
      log.retention.hours: 168
      # Retain up to 10GB per partition
      log.retention.bytes: 10737418240
      # Log segment size: 1GB
      log.segment.bytes: 1073741824
      # Log cleanup policy
      log.cleanup.policy: delete
      # Flush interval (let OS manage)
      log.flush.interval.messages: 9223372036854775807
      log.flush.interval.ms: 9223372036854775807
      # Enable log compaction for event sourcing use cases
      # log.cleanup.policy: compact  # Use this for event sourcing

    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 100Gi
          class: standard
          # Important: keep data when pod is deleted
          deleteClaim: false
```

### Configure Log Compaction for Event Store

```yaml
# compacted-topic.yaml - Log-compacted Kafka topic (event sourcing)
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: user-events
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-persistent
spec:
  partitions: 6
  replicas: 3
  config:
    # Compact - keep only latest value per key
    cleanup.policy: compact
    # Keep compacted records for 24 hours
    min.compaction.lag.ms: "86400000"
    # Delete records 7 days after being compacted away
    delete.retention.ms: "604800000"
    # Start compaction when segment is 70% full
    min.cleanable.dirty.ratio: "0.7"
```

## Section 3: NATS JetStream Persistence

```yaml
# nats-persistence-values.yaml - NATS with JetStream persistence
config:
  jetstream:
    enabled: true
    fileStore:
      enabled: true
      pvc:
        enabled: true
        size: 50Gi
        storageClassName: standard
    memStore:
      enabled: true
      maxSize: 1Gi  # In-memory cache for recent messages
```

### Create Persistent JetStream Stream

```bash
# Create a file-backed stream
kubectl exec -n messaging deployment/nats-box -- \
  nats stream create ORDERS \
  --server nats://nats.messaging.svc.cluster.local:4222 \
  --subjects "orders.*" \
  --storage file \        # File-based persistence (survives restart)
  --replicas 3 \          # 3 copies across NATS cluster
  --retention limits \
  --max-age 168h \        # 7 days
  --max-msgs 50000000 \
  --discard old

# View stream info
kubectl exec -n messaging deployment/nats-box -- \
  nats stream info ORDERS \
  --server nats://nats.messaging.svc.cluster.local:4222
```

## Section 4: Configure Longhorn Backup for Message Queue Data

```yaml
# longhorn-mq-backup.yaml - Automated backup of message queue PVCs
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: mq-hourly-snapshot
  namespace: longhorn-system
spec:
  cron: "0 * * * *"  # Every hour
  task: snapshot
  groups:
    - message-queues
  retain: 48   # Keep 48 hourly snapshots (2 days)
  concurrency: 1
---
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: mq-daily-backup
  namespace: longhorn-system
spec:
  cron: "0 3 * * *"  # Daily at 3 AM
  task: backup
  groups:
    - message-queues
  retain: 30   # Keep 30 days of backups
  concurrency: 1
```

Tag message queue PVCs for backup:

```bash
# Tag RabbitMQ PVCs for backup
for PVC in $(kubectl get pvc -n messaging -o name); do
  kubectl annotate $PVC \
    -n messaging \
    "recurring-job-group.longhorn.io/message-queues=enabled"
done
```

## Section 5: Disaster Recovery Test

```bash
#!/bin/bash
# dr-test.sh - Test message persistence after restart

NAMESPACE="messaging"
RABBITMQ_POD="rabbitmq-persistent-0"

echo "=== Publishing 1000 test messages ==="
kubectl exec -n $NAMESPACE $RABBITMQ_POD -- \
  rabbitmqadmin publish \
  exchange=amq.default \
  routing_key=test-persistence \
  payload="test message"

echo "=== Count messages before restart ==="
kubectl exec -n $NAMESPACE $RABBITMQ_POD -- \
  rabbitmqctl list_queues name messages | grep test-persistence

echo "=== Restarting RabbitMQ pod ==="
kubectl delete pod $RABBITMQ_POD -n $NAMESPACE

echo "=== Waiting for pod to restart ==="
kubectl wait pod -n $NAMESPACE \
  -l app.kubernetes.io/name=rabbitmq-persistent \
  --for=condition=Ready \
  --timeout=120s

echo "=== Count messages after restart ==="
kubectl exec -n $NAMESPACE $RABBITMQ_POD -- \
  rabbitmqctl list_queues name messages | grep test-persistence
# Messages should be preserved
```

## Conclusion

Proper persistence configuration is essential for preventing message loss in production environments. For RabbitMQ, use quorum queues which are always durable and provide better safety guarantees than classic mirrored queues. For Kafka, configure appropriate retention periods and replication factors, ensuring `deleteClaim: false` on PVCs so data survives pod deletions. Combine application-level persistence with Longhorn volume backups for complete data protection. Always test your persistence configuration by simulating broker restarts before deploying to production.
