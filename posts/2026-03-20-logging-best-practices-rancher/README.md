# How to Implement Logging Best Practices in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Logging, Loki, Fluentd, ELK, Kubernetes, Observability

Description: Implement logging best practices in Rancher using structured logging, centralized log aggregation with Loki or Elasticsearch, log retention policies, and alerting on log patterns for production...

## Introduction

Effective logging in Rancher covers container logs, Kubernetes events, audit logs, and Rancher management plane logs. Log aggregation to a central store enables searching, correlation, and alerting. The key principles are: structured JSON logs, centralized aggregation, retention policies, and alerting on error patterns.

## Step 1: Enable Rancher Logging

Install via Rancher UI: **Cluster > Apps > Charts > Logging**

```bash
# Or install via Helm

helm repo add rancher-charts https://charts.rancher.io
helm install rancher-logging rancher-charts/rancher-logging \
  --namespace cattle-logging-system \
  --create-namespace
```

Rancher Logging uses the Banzai Cloud logging operator with Fluentd/Fluentbit.

## Step 2: Configure Log Flows to Loki

```yaml
# ClusterFlow: collect all container logs
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: all-logs
  namespace: cattle-logging-system
spec:
  match:
    - select: {}           # Match all pods
  filters:
    - tag_normaliser: {}
    - parser:
        remove_key_name_field: true
        parse:
          type: json       # Parse JSON structured logs
  globalOutputRefs:
    - loki-output
---
# ClusterOutput: send to Loki
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: loki-output
  namespace: cattle-logging-system
spec:
  loki:
    url: http://loki.monitoring.svc.cluster.local:3100
    configure_kubernetes_labels: true
    labels:
      cluster: production
      app: "$.kubernetes.labels.app"
      namespace: "$.kubernetes.namespace_name"
    buffer:
      chunk_limit_size: 8MB
      flush_interval: 10s
      retry_max_times: 5
```

## Step 3: Structured Logging in Applications

Applications must emit structured JSON for effective log querying:

```json
{
  "timestamp": "2026-03-20T10:00:00Z",
  "level": "ERROR",
  "service": "payment-api",
  "trace_id": "abc123",
  "user_id": "user456",
  "message": "Payment processing failed",
  "error": "connection timeout to payment-gateway",
  "duration_ms": 5003
}
```

In Node.js (using pino):
```javascript
const logger = pino({ level: 'info' });
logger.error({
  trace_id: req.headers['x-trace-id'],
  user_id: req.user.id,
  duration_ms: elapsed
}, 'Payment processing failed');
```

## Step 4: Log Retention Policies

```yaml
# Loki retention configuration (S3 backend)
compactor:
  working_directory: /data/loki/boltdb-shipper-compactor
  shared_store: s3

# Retention per stream (table_manager approach)
table_manager:
  retention_deletes_enabled: true
  retention_period: 720h    # 30 days default

# Or per-tenant retention (Loki 2.x)
limits_config:
  retention_period: 720h    # 30 days default

# Per-stream retention via label selectors:
# Audit logs: 1 year
# Application logs: 30 days
# Debug logs: 7 days
```

## Step 5: Alert on Log Patterns

```yaml
# Loki alerting rule (requires Loki ruler)
groups:
  - name: application-errors
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({namespace="production"} |= "level=ERROR" [5m]))
          /
          sum(rate({namespace="production"} [5m])) > 0.05
        for: 2m
        annotations:
          summary: "Error rate above 5% in production namespace"

      - alert: OOMKillDetected
        expr: |
          count_over_time({app="kubernetes-events"}
            |= "OOMKilled" [5m]) > 0
        annotations:
          summary: "OOMKill event detected"
```

## Step 6: Kubernetes Audit Log Forwarding

```yaml
# Forward audit logs to separate Loki stream
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: audit-logs
spec:
  match:
    - select:
        labels:
          app.kubernetes.io/component: kube-apiserver
  globalOutputRefs:
    - loki-audit-output
```

## Logging Checklist

- Rancher Logging operator installed on all clusters
- Applications emit structured JSON logs
- Centralized aggregation to Loki or Elasticsearch
- Retention policy: 30 days production, 1 year audit
- Grafana dashboards for log exploration
- Alerting rules for error spikes and OOMKills
- Audit logs separated and retained for compliance
- Log volumes monitored to prevent storage exhaustion

## Conclusion

Production-grade logging in Rancher requires structured application logs, centralized aggregation, defined retention policies, and alerting on error patterns. The Rancher Logging app with Loki provides a lightweight, cost-effective stack. For high-volume environments, use S3-backed Loki storage and tune buffer sizes. Correlate logs with traces (Tempo) and metrics (Prometheus) for complete observability using Grafana's unified query interface.
