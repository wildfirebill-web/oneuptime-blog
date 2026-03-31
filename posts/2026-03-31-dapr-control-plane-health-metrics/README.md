# How to Monitor Dapr Control Plane Health with Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Metric, Observability, Kubernetes, Control Plane

Description: Monitor the health and performance of Dapr control plane components using Prometheus metrics to detect operator, sentry, and placement issues.

---

The Dapr control plane consists of several components: dapr-operator, dapr-sentry, dapr-placement, and dapr-dashboard. Issues in any of these can affect all services in your cluster. Monitoring their metrics gives you early warning before users are impacted.

## Dapr Control Plane Components

| Component | Role | Metrics Port |
|-----------|------|-------------|
| dapr-operator | Manages Dapr components | 9090 |
| dapr-sentry | Issues mTLS certificates | 9090 |
| dapr-placement | Coordinates actor placement | 9090 |
| dapr-dashboard | Web UI | 8080 |

## Scraping Control Plane Metrics

Add a Prometheus scrape job for the Dapr system namespace:

```yaml
scrape_configs:
  - job_name: 'dapr-control-plane'
    static_configs:
      - targets:
        - dapr-operator.dapr-system:9090
        - dapr-sentry.dapr-system:9090
        - dapr-placement-server.dapr-system:9090
    metrics_path: /metrics
    relabel_configs:
      - source_labels: [__address__]
        regex: "([^.]+)\\..*"
        target_label: component
        replacement: "$1"
```

## Key Control Plane Metrics

Check operator health:

```
# Operator reconciliation errors
rate(dapr_operator_reconcile_errors_total[5m])

# Components loaded
dapr_operator_components_loaded
```

Monitor sentry certificate issuance:

```
# Certificate issuance rate
rate(dapr_sentry_cert_sign_request_received_total[5m])

# Certificate issuance failures
rate(dapr_sentry_cert_sign_failed_total[5m])

# Certificate issuance latency
histogram_quantile(0.99, rate(dapr_sentry_cert_sign_duration_seconds_bucket[5m]))
```

Track placement health:

```
# Active placement streams (connected runtimes)
dapr_placement_runtimes_total

# Placement table dissemination time
dapr_placement_actor_heartbeat_timestamp
```

## Health Check Endpoints

Beyond metrics, check the health endpoints directly:

```bash
# Check operator health
kubectl exec -n dapr-system deploy/dapr-operator -- \
  wget -qO- http://localhost:8080/healthz

# Check sentry health
kubectl exec -n dapr-system deploy/dapr-sentry -- \
  wget -qO- http://localhost:8080/healthz

# Check placement health
kubectl exec -n dapr-system deploy/dapr-placement-server -- \
  wget -qO- http://localhost:8080/healthz
```

## Control Plane Alert Rules

```yaml
groups:
- name: dapr-control-plane
  rules:
  - alert: DaprOperatorDown
    expr: up{job="dapr-control-plane", component="dapr-operator"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Dapr operator is down"

  - alert: DaprSentryCertFailures
    expr: rate(dapr_sentry_cert_sign_failed_total[5m]) > 0
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Dapr Sentry is failing to sign certificates"

  - alert: DaprPlacementLowConnections
    expr: dapr_placement_runtimes_total < 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "No runtimes connected to Dapr placement service"
```

## Viewing Control Plane Logs

When metrics indicate an issue, check the logs:

```bash
# Operator logs
kubectl logs -n dapr-system deploy/dapr-operator --tail=100

# Sentry logs
kubectl logs -n dapr-system deploy/dapr-sentry --tail=100

# Placement logs
kubectl logs -n dapr-system statefulset/dapr-placement-server --tail=100
```

## Summary

Monitoring Dapr control plane health requires scraping metrics from the dapr-system namespace and alerting on component availability, certificate issuance failures, and placement connectivity. The sentry and operator components are particularly critical - failures there prevent new certificates from being issued and new components from being loaded. Combine metrics with health endpoint checks for a complete picture of control plane status.
