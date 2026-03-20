# How to Monitor Serverless Workloads in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, serverless, monitoring, prometheus, grafana, observability

Description: Comprehensive guide to monitoring serverless workloads in Rancher with Prometheus, Grafana, and distributed tracing.

## Introduction

Monitoring serverless workloads presents unique challenges: functions scale to zero (no persistent metrics), cold starts affect latency, and invocations may be very short-lived. This guide covers effective observability strategies for serverless on Rancher.

## Key Metrics to Monitor

- **Invocation rate**: Requests/invocations per second
- **Error rate**: Failed invocations percentage
- **Latency**: P50, P95, P99 response times
- **Cold start rate**: Percentage of cold start invocations
- **Concurrent executions**: Active function instances
- **Scale events**: Scale-up and scale-down frequency

## Step 1: Configure Knative Metrics

```yaml
# knative-metrics-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-observability
  namespace: knative-serving
data:
  metrics.backend-destination: prometheus
  metrics.request-metrics-backend-destination: prometheus
  logging.enable-var-log-collection: "true"
  logging.revision-url-template: |
    http://logging.example.com/?stream=stdout&ns=${NAMESPACE}&pod=${POD}&container=${CONTAINER}
```

## Step 2: Configure OpenFaaS Metrics

```yaml
# openfaas-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: openfaas-gateway
  namespace: openfaas
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: gateway
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

## Step 3: Install Distributed Tracing

```bash
# Install Jaeger for distributed tracing
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

helm install jaeger jaegertracing/jaeger \
  --namespace monitoring \
  --set provisionDataStore.cassandra=false \
  --set allInOne.enabled=true \
  --set storage.type=memory \
  --set agent.enabled=false \
  --set collector.enabled=false \
  --set query.enabled=false
```

```yaml
# Configure Knative to send traces to Jaeger
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-tracing
  namespace: knative-serving
data:
  backend: zipkin                          # Jaeger supports Zipkin format
  zipkin-endpoint: http://jaeger-collector.monitoring.svc.cluster.local:9411/api/v2/spans
  sample-rate: "0.1"                       # Trace 10% of requests
```

## Step 4: Prometheus Recording Rules for Serverless

```yaml
# serverless-recording-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: serverless-recording-rules
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: serverless.recording
    interval: 30s
    rules:
    # Knative RPS
    - record: knative:revision:rps
      expr: |
        sum(rate(revision_request_count{response_code_class="2xx"}[2m])) 
        by (revision_name, namespace_name)
    
    # Error rate
    - record: knative:revision:error_rate
      expr: |
        sum(rate(revision_request_count{response_code_class=~"4xx|5xx"}[2m])) 
        by (revision_name) /
        sum(rate(revision_request_count[2m])) by (revision_name)
    
    # P95 latency
    - record: knative:revision:latency_p95
      expr: |
        histogram_quantile(0.95, 
          sum(rate(revision_request_latencies_bucket[5m])) 
          by (revision_name, le))
    
    # Scale events per hour
    - record: knative:revision:scale_events_per_hour
      expr: |
        sum(increase(autoscaler_desired_pods[1h])) 
        by (revision_name)
```

## Step 5: Alert Rules for Serverless

```yaml
# serverless-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: serverless-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: serverless.alerts
    rules:
    - alert: FunctionHighErrorRate
      expr: knative:revision:error_rate > 0.05
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Function {{ $labels.revision_name }} has high error rate"
        description: "Error rate is {{ $value | humanizePercentage }}"
    
    - alert: FunctionHighLatency
      expr: knative:revision:latency_p95 > 2000
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Function {{ $labels.revision_name }} P95 latency is high"
        description: "P95 latency is {{ $value }}ms"
    
    - alert: KedaScalerError
      expr: keda_scaler_errors_total > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "KEDA scaler error for {{ $labels.scaler }}"
```

## Step 6: Grafana Dashboard for Serverless

```json
{
  "title": "Serverless Function Overview",
  "panels": [
    {
      "title": "Invocations Per Second",
      "type": "stat",
      "targets": [{"expr": "sum(knative:revision:rps) by (revision_name)"}]
    },
    {
      "title": "Error Rate %",
      "type": "gauge",
      "targets": [{"expr": "avg(knative:revision:error_rate) * 100"}],
      "fieldConfig": {"defaults": {"thresholds": {"steps": [
        {"color": "green", "value": 0},
        {"color": "yellow", "value": 1},
        {"color": "red", "value": 5}
      ]}}}
    },
    {
      "title": "P95 Latency (ms)",
      "type": "timeseries",
      "targets": [{"expr": "knative:revision:latency_p95"}]
    },
    {
      "title": "Active Replicas",
      "type": "timeseries",
      "targets": [{"expr": "autoscaler_actual_pods"}]
    }
  ]
}
```

## Step 7: Structured Logging for Functions

```python
# Structured logging in Python function
import json
import logging
import time

logger = logging.getLogger(__name__)
logging.basicConfig(
    format='%(message)s',
    level=logging.INFO
)

def handler(req):
    start_time = time.time()
    
    try:
        result = process(req)
        duration_ms = (time.time() - start_time) * 1000
        
        # Structured log for easy querying
        logger.info(json.dumps({
            "event": "function_executed",
            "status": "success",
            "duration_ms": duration_ms,
            "input_size": len(req),
            "output_size": len(result)
        }))
        
        return result
    except Exception as e:
        logger.error(json.dumps({
            "event": "function_error",
            "error": str(e),
            "duration_ms": (time.time() - start_time) * 1000
        }))
        raise
```

## Conclusion

Monitoring serverless workloads requires a multi-signal approach: metrics for performance and scaling, traces for request flow, and structured logs for debugging. With Prometheus, Grafana, and Jaeger integrated into your Rancher cluster, you gain full observability into your serverless functions' behavior, performance, and resource consumption.
