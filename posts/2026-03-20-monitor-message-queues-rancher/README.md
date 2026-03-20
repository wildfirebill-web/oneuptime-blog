# How to Monitor Message Queues in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Monitoring, Message Queues, Prometheus, Grafana, Alerting

Description: Set up comprehensive monitoring for message queue workloads in Rancher using Prometheus, Grafana dashboards, and alerting rules.

## Introduction

Monitoring message queues requires tracking queue depths, consumer lag, message rates, and broker health. Unchecked queue growth can indicate consumer failures, leading to cascading application issues. This guide sets up monitoring for RabbitMQ and Kafka using Prometheus and Grafana in Rancher.

## Step 1: Install Prometheus Stack

If not already installed, deploy the kube-prometheus-stack via Rancher Apps:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.enabled=true \
  --set prometheus.enabled=true
```

## Step 2: Enable RabbitMQ Metrics

The Bitnami RabbitMQ chart supports Prometheus metrics. Enable them in your values file:

```yaml
# Add to rabbitmq-values.yaml

metrics:
  enabled: true
  serviceMonitor:
    enabled: true          # Creates a ServiceMonitor for Prometheus Operator
    namespace: monitoring   # Namespace where Prometheus is installed
    labels:
      release: monitoring   # Must match Prometheus Operator's label selector
```

## Step 3: Enable Kafka Metrics

For Kafka, the JMX exporter is the standard approach:

```yaml
# Add to kafka-values.yaml
metrics:
  kafka:
    enabled: true
  jmx:
    enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
    labels:
      release: monitoring
```

## Step 4: Configure Prometheus AlertRules

Define alert rules to detect critical queue conditions:

```yaml
# mq-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: message-queue-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
    - name: rabbitmq
      rules:
        - alert: RabbitMQQueueDepthHigh
          expr: rabbitmq_queue_messages > 10000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "RabbitMQ queue depth is high"
            description: "Queue {{ $labels.queue }} has {{ $value }} messages"

        - alert: RabbitMQNodeDown
          expr: rabbitmq_running == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "RabbitMQ node is down"
```

```bash
kubectl apply -f mq-alerts.yaml
```

## Step 5: Import Grafana Dashboards

Import pre-built dashboards for common message queues:

- **RabbitMQ**: Dashboard ID `10991`
- **Kafka**: Dashboard ID `7589`

Access Grafana at `http://grafana.monitoring.svc.cluster.local:3000` and go to **Dashboards > Import**, then enter the dashboard ID.

## Step 6: Monitor Kafka Consumer Lag

Consumer lag is one of the most critical Kafka metrics. Track it with a dedicated query:

```promql
# PromQL: Consumer group lag across all partitions
sum by (consumergroup, topic) (
  kafka_consumergroup_lag
)
```

## Step 7: Set Up OneUptime Monitors

For external availability monitoring, create a TCP monitor in [OneUptime](https://oneuptime.com) targeting your message queue's external endpoint. This verifies connectivity independently of internal Prometheus metrics.

## Conclusion

Comprehensive message queue monitoring requires tracking broker health, queue depths, consumer lag, and message throughput. The combination of Prometheus ServiceMonitors, Grafana dashboards, and PrometheusRules provides full observability for production message queue deployments in Rancher.
