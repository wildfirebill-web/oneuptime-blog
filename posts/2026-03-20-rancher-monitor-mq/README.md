# How to Monitor Message Queues in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Message Queue, Monitoring, Prometheus, Grafana

Description: Set up comprehensive monitoring for RabbitMQ, Kafka, and other message queues in Rancher using Prometheus and Grafana dashboards.

## Introduction

Monitoring message queues is critical for ensuring message delivery, detecting backlogs, and preventing consumer lag from causing service degradation. This guide covers setting up Prometheus metrics collection and Grafana dashboards for RabbitMQ, Apache Kafka, and NATS in Rancher-managed clusters.

## Prerequisites

- Rancher Monitoring (Prometheus/Grafana) stack installed
- Message queue deployments with metrics endpoints
- kubectl access

## Step 1: Enable Metrics for RabbitMQ

```yaml
# rabbitmq-with-metrics.yaml - RabbitMQ with Prometheus plugin
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-monitored
  namespace: messaging
spec:
  replicas: 3
  rabbitmq:
    additionalPlugins:
      # Enable Prometheus metrics plugin
      - rabbitmq_prometheus
    additionalConfig: |
      # Detailed metrics for per-queue stats
      prometheus.return_per_object_metrics = true
      prometheus.path = /metrics
```

```yaml
# rabbitmq-servicemonitor.yaml - ServiceMonitor for RabbitMQ
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rabbitmq-metrics
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  namespaceSelector:
    matchNames:
      - messaging
  selector:
    matchLabels:
      app.kubernetes.io/component: rabbitmq
  endpoints:
    - port: prometheus
      interval: 30s
      path: /metrics
```

## Step 2: Enable Metrics for Kafka (Strimzi)

```yaml
# kafka-with-metrics.yaml - Kafka cluster with JMX metrics
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka-monitored
  namespace: kafka
spec:
  kafka:
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-jmx-config
          key: kafka-metrics-config.yml
    replicas: 3
    storage:
      type: ephemeral

  zookeeper:
    replicas: 3
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-jmx-config
          key: zookeeper-metrics-config.yml
    storage:
      type: ephemeral

  kafkaExporter:
    topicRegex: ".*"
    groupRegex: ".*"
---
# kafka-jmx-config.yaml - JMX metrics configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-jmx-config
  namespace: kafka
data:
  kafka-metrics-config.yml: |
    lowercaseOutputName: true
    lowercaseOutputLabelNames: true
    rules:
      - pattern: "kafka.server<type=BrokerTopicMetrics, name=MessagesInPerSec, topic=(.+)><>Count"
        name: kafka_server_broker_topic_messages_in_total
        labels:
          topic: "$1"
        type: COUNTER
      - pattern: "kafka.server<type=BrokerTopicMetrics, name=BytesInPerSec, topic=(.+)><>Count"
        name: kafka_server_broker_topic_bytes_in_total
        labels:
          topic: "$1"
        type: COUNTER
      - pattern: "kafka.server<type=ReplicaManager, name=(.+)><>Value"
        name: kafka_server_replica_manager_$1
        type: GAUGE
      - pattern: "kafka.controller<type=KafkaController, name=(.+)><>Value"
        name: kafka_controller_$1
        type: GAUGE
  zookeeper-metrics-config.yml: |
    lowercaseOutputName: true
    rules:
      - pattern: "org.apache.ZooKeeperService<name0=ReplicatedServer_id(\\d+)><>(\\w+)"
        name: "zookeeper_$2"
        type: GAUGE
```

```yaml
# kafka-podmonitor.yaml - PodMonitor for Kafka (Strimzi creates pods)
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: kafka-metrics
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  namespaceSelector:
    matchNames:
      - kafka
  selector:
    matchLabels:
      strimzi.io/kind: Kafka
  podMetricsEndpoints:
    - port: tcp-prometheus
      interval: 30s
      path: /metrics
```

## Step 3: Configure Prometheus Alerting Rules

```yaml
# mq-alerts.yaml - Comprehensive message queue alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: message-queue-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: rabbitmq
      rules:
        # RabbitMQ - Queue depth alert
        - alert: RabbitMQQueueDepthHigh
          expr: rabbitmq_queue_messages > 10000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "RabbitMQ queue {{ $labels.queue }} has {{ $value }} messages"

        # RabbitMQ - No consumers
        - alert: RabbitMQQueueNoConsumers
          expr: rabbitmq_queue_consumers == 0 and rabbitmq_queue_messages > 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "RabbitMQ queue {{ $labels.queue }} has no consumers but has {{ $value | humanize }} messages"

        # RabbitMQ - Memory high
        - alert: RabbitMQMemoryHigh
          expr: rabbitmq_process_resident_memory_bytes / rabbitmq_resident_memory_limit_bytes > 0.8
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "RabbitMQ memory usage is {{ $value | humanizePercentage }}"

    - name: kafka
      rules:
        # Kafka consumer lag
        - alert: KafkaConsumerGroupLagHigh
          expr: kafka_consumergroup_lag > 1000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Kafka consumer group {{ $labels.consumergroup }} topic {{ $labels.topic }} partition {{ $labels.partition }} lag is {{ $value }}"

        # Kafka under-replicated partitions
        - alert: KafkaUnderReplicatedPartitions
          expr: kafka_server_ReplicaManager_UnderReplicatedPartitions > 0
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "Kafka broker {{ $labels.instance }} has {{ $value }} under-replicated partitions"

        # Kafka no active controller
        - alert: KafkaNoActiveController
          expr: sum(kafka_controller_KafkaController_ActiveControllerCount) by (job) == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "No active Kafka controller found"
```

## Step 4: Import Grafana Dashboards

```bash
# Import RabbitMQ dashboard
# Dashboard ID: 10991 (RabbitMQ Overview)
# Dashboard ID: 4279 (RabbitMQ + Kubernetes)

# Import Kafka dashboards
# Dashboard ID: 7589 (Kafka Overview)
# Dashboard ID: 7592 (Kafka Consumer Groups)

# Import NATS dashboard
# Dashboard ID: 2279 (NATS Server)

# Download and apply as ConfigMaps
curl -s "https://grafana.com/api/dashboards/10991/revisions/1/download" | \
  kubectl create configmap grafana-dashboard-rabbitmq \
  --from-file=dashboard.json=/dev/stdin \
  --namespace=cattle-monitoring-system \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Step 5: Create a Kafka Consumer Lag Dashboard

```bash
# Check consumer lag from the command line
kubectl exec -n kafka kafka-monitored-kafka-0 -- \
  bin/kafka-consumer-groups.sh \
  --bootstrap-server kafka-monitored-kafka-bootstrap:9092 \
  --describe \
  --all-groups | \
  sort -k6 -rn | head -20
```

## Step 6: Real-Time Queue Monitoring Script

```bash
#!/bin/bash
# monitor-queues.sh - Real-time queue monitoring

RABBITMQ_NS="messaging"
KAFKA_NS="kafka"

echo "=== RabbitMQ Queue Status ==="
kubectl exec -n $RABBITMQ_NS \
  $(kubectl get pod -n $RABBITMQ_NS -l app.kubernetes.io/component=rabbitmq -o name | head -1) -- \
  rabbitmqctl list_queues -p / name messages consumers messages_unacknowledged 2>/dev/null

echo ""
echo "=== Kafka Consumer Group Lag ==="
kubectl exec -n $KAFKA_NS \
  $(kubectl get pod -n $KAFKA_NS -l strimzi.io/name=kafka-monitored-kafka -o name | head -1) -- \
  bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list 2>/dev/null | while read GROUP; do
  echo "Group: $GROUP"
  kubectl exec -n $KAFKA_NS \
    $(kubectl get pod -n $KAFKA_NS -l strimzi.io/name=kafka-monitored-kafka -o name | head -1) -- \
    bin/kafka-consumer-groups.sh \
    --bootstrap-server localhost:9092 \
    --describe --group $GROUP 2>/dev/null | \
    awk 'NR>1 {sum += $6} END {print "  Total lag: " sum}'
done
```

## Conclusion

Comprehensive monitoring of message queues in Rancher prevents message loss, consumer lag, and queue buildup from going undetected until they cause service outages. The combination of Prometheus metrics, Grafana dashboards, and well-tuned alerting rules gives you complete visibility into your messaging infrastructure. Key metrics to watch include queue depth, consumer count, consumer lag, and memory/disk usage, with alerts configured to trigger before these metrics reach critical thresholds.
