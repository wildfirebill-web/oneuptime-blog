# How to Monitor Actor Performance in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Monitoring, Prometheus, Metrics

Description: Monitor Dapr actor performance using Prometheus metrics and Grafana dashboards to track active actor counts, method latency, and state store operation rates.

---

Dapr exposes rich Prometheus metrics for actor runtime performance. Setting up proper monitoring helps you detect activation storms, latency spikes, and state store bottlenecks before they impact production workloads.

## Enabling Metrics

Dapr exposes metrics on port 9090 by default. In Kubernetes, annotate your pod:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "counter-service"
  dapr.io/metrics-port: "9090"
```

Verify metrics are available:

```bash
curl http://localhost:9090/metrics | grep dapr_actor
```

## Key Actor Metrics

### Active Actor Count

```text
dapr_actor_active_actors{app_id="counter-service",actor_type="Counter",namespace="default"}
```

High counts indicate actors are not being deactivated. Check your idle timeout configuration.

### Method Invocation Latency

```text
dapr_actor_method_duration_bucket{app_id="counter-service",actor_type="Counter",method="Increment"}
```

Use this histogram to identify slow actor methods.

### Activation and Deactivation Rate

```text
dapr_actor_activations_total{actor_type="Counter"}
dapr_actor_deactivations_total{actor_type="Counter"}
```

A high activation rate with low deactivation rate signals actors are accumulating in memory.

## Prometheus Scrape Configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: "dapr-actor-service"
    static_configs:
      - targets: ["counter-service-pod:9090"]
    metrics_path: /metrics
```

## Grafana Dashboard Queries

### Active Actors Over Time

```promql
sum(dapr_actor_active_actors{app_id="counter-service"}) by (actor_type)
```

### P99 Method Latency

```promql
histogram_quantile(0.99,
  sum(rate(dapr_actor_method_duration_bucket{actor_type="Counter"}[5m])) by (le, method)
)
```

### Activation Rate Per Minute

```promql
sum(rate(dapr_actor_activations_total{actor_type="Counter"}[1m])) by (actor_type)
```

## Setting Up Alerts

Alert when active actor count grows unboundedly:

```yaml
# alertmanager rule
groups:
- name: dapr-actors
  rules:
  - alert: ActorCountTooHigh
    expr: dapr_actor_active_actors{actor_type="Counter"} > 10000
    for: 5m
    annotations:
      summary: "Actor count exceeds threshold"
      description: "Counter actor count is {{ $value }}, check idle timeout settings"
```

Alert on high method latency:

```yaml
  - alert: ActorMethodLatencyHigh
    expr: |
      histogram_quantile(0.99,
        sum(rate(dapr_actor_method_duration_bucket[5m])) by (le, actor_type, method)
      ) > 1.0
    for: 2m
    annotations:
      summary: "Actor method P99 latency exceeds 1 second"
```

## Correlating with Application Traces

Cross-reference Prometheus metrics with Zipkin or Jaeger traces by correlating timestamps. When latency spikes, look for corresponding distributed traces with `dapr.actor` span tags to find the slow code path.

## Summary

Dapr's built-in Prometheus metrics provide everything needed to monitor actor performance at scale. Tracking active actor counts, method latency histograms, and activation rates gives you early warning of configuration issues and performance bottlenecks. Combining metrics with distributed tracing provides complete observability for production actor-based systems.
