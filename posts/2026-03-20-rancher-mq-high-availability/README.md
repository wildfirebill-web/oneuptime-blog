# How to Configure Message Queue High Availability in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Message Queue, High Availability, RabbitMQ, Kafka

Description: Configure high availability for message queue deployments in Rancher using quorum queues, replication, and pod anti-affinity rules to ensure zero message loss.

## Introduction

Message queue high availability ensures that your messaging infrastructure remains operational even when individual nodes fail. Different message queue systems provide different HA mechanisms: RabbitMQ uses Quorum Queues and mirroring, Kafka uses partition replication, and NATS JetStream uses stream replication. This guide covers implementing HA for the most common message queues on Rancher.

## Prerequisites

- Rancher-managed cluster with at least 3 nodes
- Message queue deployments (RabbitMQ, Kafka, or NATS)
- kubectl access

## Section 1: RabbitMQ High Availability

### Configure Quorum Queues (Recommended)

Quorum queues provide Raft-based consensus for guaranteed delivery:

```yaml
# rabbitmq-ha-cluster.yaml - High availability RabbitMQ
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-ha
  namespace: messaging
spec:
  replicas: 3

  rabbitmq:
    additionalConfig: |
      # Use quorum queues by default
      queue.default_queue_type = quorum

      # Quorum queue settings
      quorum_queue.initial_cluster_size.min = 3

      # Prevent minority partitions from accepting writes
      cluster_partition_handling = pause_minority

      # Network partition detection
      net_ticktime = 60

  # Spread across nodes using anti-affinity
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: rabbitmq-ha
          topologyKey: kubernetes.io/hostname

  # Tolerate node failures
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: rabbitmq-ha
```

### Create Quorum Queue with API

```bash
# Create quorum queue with proper HA settings
curl -s -u admin:AdminP@ss \
  -X PUT \
  -H "Content-Type: application/json" \
  http://localhost:15672/api/queues/%2F/orders-ha \
  -d '{
    "durable": true,
    "arguments": {
      "x-queue-type": "quorum",
      "x-quorum-initial-group-size": 3,
      "x-delivery-limit": 5
    }
  }'
```

## Section 2: Apache Kafka High Availability

### Configure Kafka Replication

```yaml
# kafka-ha-cluster.yaml - Kafka with HA configuration
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka-ha
  namespace: kafka
spec:
  kafka:
    version: 3.7.0
    replicas: 3
    config:
      # Minimum in-sync replicas for durability
      min.insync.replicas: 2
      # Replication factors for internal topics
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      # Leader election timeout
      leader.imbalance.check.interval.seconds: 300
      # Unclean leader election (set to false for safety)
      unclean.leader.election.enable: "false"
      # Log replication
      log.recovery.threads.per.data.dir: 2

    # Anti-affinity to spread brokers across nodes
    template:
      pod:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchLabels:
                    strimzi.io/name: kafka-ha-kafka
                topologyKey: kubernetes.io/hostname

    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 100Gi
          class: standard
          deleteClaim: false

  zookeeper:
    replicas: 3
    template:
      pod:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchLabels:
                    strimzi.io/name: kafka-ha-zookeeper
                topologyKey: kubernetes.io/hostname
```

### Create HA Topics

```yaml
# kafka-ha-topic.yaml - Highly available Kafka topic
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders-ha
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-ha
spec:
  partitions: 12
  replicas: 3        # All 3 brokers hold a copy
  config:
    # Require all in-sync replicas to acknowledge
    min.insync.replicas: "2"
    # Keep messages for 7 days
    retention.ms: "604800000"
    # Disable unclean leader election for this topic
    unclean.leader.election.enable: "false"
```

## Section 3: NATS JetStream High Availability

```yaml
# nats-ha-values.yaml - NATS cluster with JetStream HA
config:
  cluster:
    enabled: true
    replicas: 3

  jetstream:
    enabled: true
    fileStore:
      enabled: true
      pvc:
        enabled: true
        size: 20Gi

# Anti-affinity for NATS pods
podAntiAffinity:
  required:
    - topologyKey: kubernetes.io/hostname
```

Configure HA stream:

```bash
# Create a stream with replication factor 3
nats stream create ORDERS \
  --subjects "orders.*" \
  --storage file \
  --replicas 3 \
  --retention limits \
  --max-age 7d \
  --server nats://nats.messaging.svc.cluster.local:4222
```

## Section 4: Cross-Region Message Queue HA

```yaml
# cross-region-ha.yaml - Multi-region message queue setup
# RabbitMQ Federation for cross-cluster HA
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-region-a
  namespace: messaging
spec:
  replicas: 3
  rabbitmq:
    additionalPlugins:
      - rabbitmq_federation
      - rabbitmq_federation_management
    additionalConfig: |
      # Configure federation to replicate critical queues
      # to region B for disaster recovery
```

## Section 5: Monitoring Queue Health

```yaml
# mq-ha-alerts.yaml - Alerts for message queue HA health
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: mq-ha-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: rabbitmq-ha
      rules:
        # Alert if quorum is lost
        - alert: RabbitMQQuorumLost
          expr: |
            rabbitmq_queue_quorum_votes < 2
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "RabbitMQ quorum queue has lost quorum"

        # Alert if a node goes down
        - alert: RabbitMQNodeDown
          expr: |
            rabbitmq_identity_info == 0
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "RabbitMQ node {{ $labels.instance }} is down"

    - name: kafka-ha
      rules:
        # Alert if under-replicated partitions exist
        - alert: KafkaUnderReplicatedPartitions
          expr: |
            kafka_server_ReplicaManager_UnderReplicatedPartitions > 0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Kafka has under-replicated partitions"

        # Alert if offline partitions exist
        - alert: KafkaOfflinePartitions
          expr: |
            kafka_controller_KafkaController_OfflinePartitionsCount > 0
          for: 0s
          labels:
            severity: critical
          annotations:
            summary: "Kafka has offline partitions"
```

## Section 6: Pod Disruption Budgets

```yaml
# pdb-mq.yaml - Ensure minimum availability during maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: rabbitmq-pdb
  namespace: messaging
spec:
  # At least 2 of 3 RabbitMQ nodes must remain available
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: rabbitmq-ha
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: kafka-pdb
  namespace: kafka
spec:
  minAvailable: 2
  selector:
    matchLabels:
      strimzi.io/name: kafka-ha-kafka
```

## Conclusion

Message queue high availability on Rancher requires a combination of application-level replication (quorum queues, Kafka replication), Kubernetes scheduling controls (pod anti-affinity), and Pod Disruption Budgets. Always test your HA configuration by simulating node failures before relying on it in production. Monitor queue depths, replication lag, and node health, and set up alerting for conditions like lost quorum or under-replicated partitions that indicate your HA guarantees are at risk.
