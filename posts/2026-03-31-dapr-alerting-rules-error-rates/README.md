# How to Create Alerting Rules for Dapr Error Rates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Alerting, Error Rate, Prometheus, Reliability

Description: Build Prometheus alerting rules to detect elevated error rates in Dapr service invocation, state management, and pub/sub operations.

---

Error rate alerting is fundamental to site reliability engineering. Dapr's Prometheus metrics expose error counters for every building block, enabling precise alerting rules that catch reliability degradation before it breaches your SLOs. This guide walks through creating effective error rate alerts for all major Dapr operations.

## Dapr Error Rate Metrics

Dapr tracks success and failure counts for each operation type:

- `dapr_http_server_request_count` - HTTP request counts with status codes
- `dapr_service_invocation_req_sent_total` - Service invocation by status
- `dapr_state_get_total` / `dapr_state_set_total` - State store ops with success labels
- `dapr_pubsub_publish_count` - Pub/sub publish attempts with success label

## Core Error Rate Alerting Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dapr-error-rate-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
    - name: dapr.error.rates
      interval: 30s
      rules:
        - alert: DaprHighServiceInvocationErrorRate
          expr: |
            sum(rate(dapr_http_server_request_count{status=~"5.."}[5m])) by (app_id)
            /
            sum(rate(dapr_http_server_request_count[5m])) by (app_id)
            > 0.05
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Dapr service invocation error rate above 5%"
            description: "App {{ $labels.app_id }} has {{ $value | humanizePercentage }} error rate over last 5 minutes."

        - alert: DaprStateStoreWriteErrorRate
          expr: |
            rate(dapr_state_set_total{success="false"}[5m])
            /
            rate(dapr_state_set_total[5m])
            > 0.01
          for: 3m
          labels:
            severity: warning
          annotations:
            summary: "Dapr state store write errors elevated"
            description: "State store {{ $labels.storeName }} write error rate: {{ $value | humanizePercentage }}."

        - alert: DaprPubSubDeliveryErrorRate
          expr: |
            rate(dapr_pubsub_publish_count{success="false"}[5m])
            /
            rate(dapr_pubsub_publish_count[5m])
            > 0.02
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Dapr pub/sub publish error rate elevated"
            description: "Pub/sub component {{ $labels.component }} error rate: {{ $value | humanizePercentage }}."

        - alert: DaprCriticalErrorRate
          expr: |
            sum(rate(dapr_http_server_request_count{status=~"5.."}[5m])) by (app_id)
            /
            sum(rate(dapr_http_server_request_count[5m])) by (app_id)
            > 0.20
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Dapr critical error rate - immediate action required"
            description: "App {{ $labels.app_id }} error rate is {{ $value | humanizePercentage }}, far above threshold."
```

## Adding Burn Rate Alerts for SLO Protection

Multi-window burn rate alerts catch both fast and slow error rate degradation:

```yaml
        - alert: DaprSLOBurnRateFast
          expr: |
            (
              sum(rate(dapr_http_server_request_count{status=~"5.."}[1h])) by (app_id)
              /
              sum(rate(dapr_http_server_request_count[1h])) by (app_id)
            ) > (14.4 * 0.01)
          for: 2m
          labels:
            severity: critical
            page: "true"
          annotations:
            summary: "Dapr fast burn rate - SLO budget exhausting rapidly"
            description: "App {{ $labels.app_id }} is burning SLO error budget 14x faster than normal."
```

## Testing Error Rate Alerts

Inject errors by temporarily returning HTTP 500 from a service:

```bash
# Apply a fault injection config via Dapr resiliency
kubectl apply -f - <<EOF
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: test-fault
spec:
  policies:
    circuitBreakers:
      simpleCB:
        maxRequests: 1
        timeout: 10s
        trip: consecutiveFailures > 1
EOF
```

Query error rates directly in Prometheus:

```bash
curl 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=rate(dapr_http_server_request_count{status=~"5.."}[5m])'
```

## Summary

Dapr error rate alerting rules use ratio-based expressions to detect reliability degradation across service invocation, state stores, and pub/sub. Combining warning thresholds with multi-window burn rate alerts provides both immediate notification and early warning of slow SLO budget consumption.
