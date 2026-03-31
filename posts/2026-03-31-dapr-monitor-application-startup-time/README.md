# How to Monitor Dapr Application Startup Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Monitoring, Observability, Performance, Kubernetes

Description: Learn how to monitor and optimize Dapr application startup time using metrics, logs, and tracing to identify bottlenecks in sidecar initialization.

---

## Why Startup Time Matters

Slow application startup affects deployment rollouts, auto-scaling responsiveness, and overall availability. With Dapr, both the application container and the Dapr sidecar must initialize before traffic is served. Understanding where time is spent during startup helps you reduce cold start latency.

## Metrics to Track

Dapr exposes Prometheus metrics for sidecar initialization. The key metric is `dapr_runtime_init_total` along with component initialization durations.

Enable metrics in your Dapr configuration:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  metric:
    enabled: true
  tracing:
    samplingRate: "1"
```

## Scraping Startup Metrics with Prometheus

Annotate your pods so Prometheus scrapes the Dapr metrics port:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"
  prometheus.io/path: "/metrics"
```

Then query startup duration in PromQL:

```bash
# Average sidecar init time per app
avg by (app_id) (dapr_runtime_init_total)

# Component load duration
dapr_component_init_total{success="true"}
```

## Using Zipkin Traces for Startup Analysis

Configure distributed tracing to capture the full startup sequence:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: tracing-config
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: http://zipkin.monitoring:9411/api/v2/spans
```

After deploying, look for spans labeled `dapr.runtime/init` in your Zipkin UI to pinpoint which component takes the longest to initialize.

## Checking Logs for Slow Component Init

Stream Dapr sidecar logs during startup:

```bash
kubectl logs -l app=myservice -c daprd --since=2m | grep -i "init\|component\|ready"
```

A healthy output looks like:

```
time="2026-03-31T10:00:01Z" level=info msg="component loaded" name=statestore type=state.redis
time="2026-03-31T10:00:02Z" level=info msg="dapr initialized. Status: Running"
```

If a component hangs (e.g., cannot reach Redis), the sidecar will block until the timeout.

## Tuning Init Timeouts

Set per-component timeouts in your Dapr component YAML to fail fast:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  initTimeout: 5s
  metadata:
    - name: redisHost
      value: redis-master:6379
```

Reduce `initTimeout` from the default 30s if you want faster failure detection.

## Kubernetes Liveness and Readiness Probes

Combine Dapr readiness with Kubernetes probes to gate traffic:

```yaml
readinessProbe:
  httpGet:
    path: /v1.0/healthz
    port: 3500
  initialDelaySeconds: 5
  periodSeconds: 3
```

This ensures your app only receives traffic once the Dapr sidecar reports healthy.

## Summary

Monitoring Dapr application startup time involves scraping Prometheus metrics from the sidecar, analyzing Zipkin traces for component init spans, and reviewing sidecar logs for slow component loading. Setting per-component `initTimeout` values and tuning Kubernetes readiness probes ensures that slow startups are detected and handled gracefully.
