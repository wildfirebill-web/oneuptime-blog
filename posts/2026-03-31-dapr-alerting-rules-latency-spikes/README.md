# How to Create Alerting Rules for Dapr Latency Spikes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Alerting, Latency, Prometheus, Observability

Description: Configure Prometheus alerting rules to detect latency spikes in Dapr service invocation, state operations, and pub/sub messaging.

---

Latency spikes in distributed systems are often the first sign of downstream degradation. Dapr exposes histogram metrics for every operation, making it straightforward to build p99 latency alerts with Prometheus alerting rules. Catching spikes early prevents cascading failures.

## Key Dapr Latency Metrics

Dapr instruments all major operations with histogram metrics:

| Metric | Operation |
|--------|-----------|
| `dapr_http_server_request_duration_msec` | Inbound HTTP requests to sidecar |
| `dapr_service_invocation_req_sent_total` | Service-to-service calls |
| `dapr_state_get_duration_msec` | State store read latency |
| `dapr_pubsub_publish_duration_msec` | Pub/sub publish latency |

## Creating Latency Alerting Rules

Deploy a PrometheusRule with histogram-based p99 calculations:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dapr-latency-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
    - name: dapr.latency
      interval: 30s
      rules:
        - alert: DaprServiceInvocationHighLatency
          expr: |
            histogram_quantile(0.99,
              rate(dapr_http_server_request_duration_msec_bucket[5m])
            ) > 500
          for: 3m
          labels:
            severity: warning
          annotations:
            summary: "Dapr service invocation p99 latency high"
            description: "App {{ $labels.app_id }} p99 latency is {{ $value }}ms, exceeding 500ms threshold."

        - alert: DaprStateStoreHighLatency
          expr: |
            histogram_quantile(0.99,
              rate(dapr_state_get_duration_msec_bucket[5m])
            ) > 200
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Dapr state store read latency spike"
            description: "State store {{ $labels.storeName }} p99 read latency is {{ $value }}ms."

        - alert: DaprPubSubPublishHighLatency
          expr: |
            histogram_quantile(0.99,
              rate(dapr_pubsub_publish_duration_msec_bucket[5m])
            ) > 300
          for: 3m
          labels:
            severity: warning
          annotations:
            summary: "Dapr pub/sub publish latency spike"
            description: "Component {{ $labels.component }} topic {{ $labels.topic }} p99 publish latency is {{ $value }}ms."
```

## Recording Rules for Efficiency

Pre-compute p99 values with recording rules to reduce query load:

```yaml
        - record: dapr:service_invocation:p99_latency_5m
          expr: |
            histogram_quantile(0.99,
              sum(rate(dapr_http_server_request_duration_msec_bucket[5m]))
              by (app_id, le)
            )

        - record: dapr:state:p99_get_latency_5m
          expr: |
            histogram_quantile(0.99,
              sum(rate(dapr_state_get_duration_msec_bucket[5m]))
              by (storeName, le)
            )
```

Reference recording rules in alerts for better dashboard performance:

```yaml
        - alert: DaprCriticalLatency
          expr: dapr:service_invocation:p99_latency_5m > 2000
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Dapr critical latency - SLO at risk"
            description: "App {{ $labels.app_id }} p99 latency exceeded 2s for 1 minute."
```

## Validating and Testing

Test rule syntax locally:

```bash
promtool check rules dapr-latency-alerts.yaml
```

Generate synthetic load to trigger latency alerts:

```bash
# Use hey or k6 to generate load
k6 run --vus 100 --duration 60s load-test.js
```

Watch metrics in real-time with kubectl:

```bash
kubectl port-forward svc/dapr-sidecar-injector 9090:9090 -n dapr-system
curl -s http://localhost:9090/metrics | grep dapr_http_server_request_duration
```

## Summary

Dapr latency alerting rules use Prometheus histogram quantiles to detect p99 spikes across service invocation, state stores, and pub/sub. Pairing recording rules with threshold alerts reduces query overhead while ensuring latency SLOs are enforced automatically.
