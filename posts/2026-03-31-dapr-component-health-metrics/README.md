# How to Monitor Dapr Component Health with Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Component, Metric, Observability, Prometheus

Description: Use Dapr component metrics to track the health of state stores, pub/sub brokers, and bindings to detect failures before they impact services.

---

Dapr components - state stores, pub/sub brokers, secret stores, and bindings - are the backing infrastructure your services depend on. When a component becomes unhealthy, the effects cascade. Dapr exposes metrics for each component type so you can detect issues early.

## Component Metric Categories

Dapr emits component metrics with the label `component_type` and `component_name`. The main categories are:

- State store: `dapr_component_state_*`
- Pub/sub: `dapr_component_pubsub_*`
- Bindings: `dapr_component_bindings_*`
- Secret store: implicit via error counts

## State Store Health Metrics

```
# GET operation error rate
rate(dapr_component_state_get_failed_total[5m])

# SET operation error rate
rate(dapr_component_state_set_failed_total[5m])

# GET latency P99
histogram_quantile(0.99,
  sum by (le, component) (
    rate(dapr_component_state_get_latencies_ms_bucket[5m])
  )
)

# Total operations by component
rate(dapr_component_state_get_total[5m])
```

## Pub/Sub Component Health

```
# Messages failing to be published
rate(dapr_component_pubsub_egress_fail_count[5m])

# Messages failing to be processed (ingress errors)
rate(dapr_component_pubsub_ingress_fail_count[5m])

# Drop rate - messages dropped due to full queues
rate(dapr_component_pubsub_drop_count[5m])

# End-to-end pub/sub latency
histogram_quantile(0.95,
  sum by (le, component, topic) (
    rate(dapr_component_pubsub_ingress_latencies_ms_bucket[5m])
  )
)
```

## Binding Component Health

```
# Binding invocation errors
rate(dapr_component_bindings_output_failed_total[5m])

# Binding invocation latency
histogram_quantile(0.99,
  sum by (le, component) (
    rate(dapr_component_bindings_output_latency_ms_bucket[5m])
  )
)
```

## Alert Rules for Component Health

```yaml
groups:
- name: dapr-components
  rules:
  - alert: DaprStateStoreErrors
    expr: rate(dapr_component_state_get_failed_total[5m]) > 0
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "State store {{ $labels.component }} has GET failures"

  - alert: DaprPubSubDropping
    expr: rate(dapr_component_pubsub_drop_count[5m]) > 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Pub/sub component {{ $labels.component }} is dropping messages on topic {{ $labels.topic }}"

  - alert: DaprPubSubEgressErrors
    expr: |
      rate(dapr_component_pubsub_egress_fail_count[5m])
        / rate(dapr_component_pubsub_egress_count[5m]) > 0.01
    for: 3m
    labels:
      severity: warning
    annotations:
      summary: "Pub/sub publish error rate above 1% on {{ $labels.component }}"
```

## Checking Component Status via Dapr API

Beyond metrics, use the Dapr health API to check component status:

```bash
# Check all component health
curl http://localhost:3500/v1.0/healthz

# Check specific component initialization
kubectl logs deploy/order-service -c daprd | grep -i "component"
```

## Grafana Dashboard for Component Health

Create a heatmap panel showing error rates across all components:

```
# Matrix query - error rate per component
sum by (component, component_type) (
  rate(dapr_component_state_get_failed_total[5m])
  + rate(dapr_component_state_set_failed_total[5m])
)
```

Use the "Table" visualization with color thresholds: green at 0, yellow above 0.01, red above 0.1.

## Summary

Dapr component health metrics cover state stores, pub/sub, and bindings with operation counts, error rates, and latency histograms. Alert on any non-zero error rates for state operations and on drop rates for pub/sub components, as drops indicate the system is falling behind. Combine metrics with the Dapr health API and sidecar logs for complete component observability.
