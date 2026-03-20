# How to Monitor Kubewarden Policy Decisions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, Monitoring, Observability

Description: Learn how to monitor Kubewarden policy decisions using OpenTelemetry, Prometheus metrics, and audit logging to gain visibility into your admission control activity.

## Introduction

Monitoring Kubewarden policy decisions is essential for understanding your security posture, identifying policy violations, debugging unexpected denials, and auditing compliance. Kubewarden provides multiple observability mechanisms: Prometheus metrics, OpenTelemetry tracing, and Kubernetes audit log integration.

This guide covers setting up comprehensive monitoring for Kubewarden policy decisions.

## Prerequisites

- Kubewarden installed and running
- Prometheus Operator or monitoring stack (optional)
- OpenTelemetry collector (optional for tracing)
- `kubectl` access to your cluster

## Kubewarden Observability Features

Kubewarden exposes:
1. **Prometheus metrics**: Counters for accepted/rejected requests per policy
2. **OpenTelemetry traces**: Distributed tracing for policy evaluation
3. **Kubernetes events**: Policy violation events in the cluster event log
4. **Audit log integration**: Via standard Kubernetes audit logging

## Configuring Prometheus Metrics

### Enabling Metrics on PolicyServer

```yaml
# policyserver-with-metrics.yaml
apiVersion: policies.kubewarden.io/v1
kind: PolicyServer
metadata:
  name: default
  namespace: kubewarden
spec:
  image: ghcr.io/kubewarden/policy-server:v1.10.0
  replicas: 2

  # Enable Prometheus metrics export
  serviceAccountName: kubewarden-policy-server-default
```

### Creating a ServiceMonitor

```yaml
# kubewarden-service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kubewarden-policy-server
  namespace: kubewarden
  labels:
    # Match your Prometheus operator's selector
    release: prometheus-stack
spec:
  selector:
    matchLabels:
      app: kubewarden-policy-server-default
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
```

### Key Kubewarden Metrics

```bash
# View available Kubewarden metrics
kubectl port-forward -n kubewarden \
  $(kubectl get pods -n kubewarden -l app=kubewarden-policy-server-default \
    -o jsonpath='{.items[0].metadata.name}') \
  8080:3000

# Access metrics endpoint
curl http://localhost:8080/metrics | grep kubewarden
```

Key metrics to monitor:
- `kubewarden_policy_evaluations_total`: Total policy evaluations by policy and result
- `kubewarden_policy_evaluation_duration_seconds`: Duration of policy evaluations
- `kubewarden_policy_evaluations_reused_total`: Cache hit rate
- `kubewarden_admission_webhook_latency_seconds`: End-to-end webhook latency

## Configuring OpenTelemetry Tracing

```yaml
# policyserver-otel.yaml
apiVersion: policies.kubewarden.io/v1
kind: PolicyServer
metadata:
  name: default
  namespace: kubewarden
spec:
  image: ghcr.io/kubewarden/policy-server:v1.10.0
  replicas: 2
  env:
    # Enable OpenTelemetry tracing
    - name: KUBEWARDEN_LOG_LEVEL
      value: "info"
    - name: KUBEWARDEN_LOG_FMT
      value: "otlp"
    # OTLP endpoint (OpenTelemetry collector)
    - name: OTEL_EXPORTER_OTLP_ENDPOINT
      value: "http://otel-collector.observability:4317"
    - name: OTEL_SERVICE_NAME
      value: "kubewarden-policy-server"
```

### Installing OpenTelemetry Collector

```yaml
# otel-collector.yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: kubewarden
  namespace: observability
spec:
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: "0.0.0.0:4317"
          http:
            endpoint: "0.0.0.0:4318"

    processors:
      batch:
        timeout: 1s

    exporters:
      # Export to Jaeger
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true
      # Export to Zipkin
      zipkin:
        endpoint: http://zipkin:9411/api/v2/spans
      # Log to stdout for debugging
      logging:
        verbosity: detailed

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [jaeger, logging]
```

## Monitoring Policy Violations via Kubernetes Events

Kubewarden records policy violations as Kubernetes events:

```bash
# Watch for policy violations in real-time
kubectl get events -A \
  --field-selector reason=PolicyViolation \
  -w

# Get recent violations with details
kubectl get events -A \
  --field-selector reason=PolicyViolation \
  --sort-by='.lastTimestamp' \
  -o custom-columns=\
'TIME:.lastTimestamp,\
NAMESPACE:.metadata.namespace,\
NAME:.involvedObject.name,\
MESSAGE:.message'

# Get violations for a specific policy
kubectl get events -A \
  -o jsonpath='{range .items[?(@.reason=="PolicyViolation")]}{.lastTimestamp} {.involvedObject.name}: {.message}{"\n"}{end}'
```

## Creating Prometheus Alerts for Policy Violations

```yaml
# kubewarden-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubewarden-policy-alerts
  namespace: kubewarden
  labels:
    release: prometheus-stack
spec:
  groups:
    - name: kubewarden.rules
      interval: 30s
      rules:
        # Alert on high policy denial rate
        - alert: KubewardenHighDenialRate
          expr: |
            rate(kubewarden_policy_evaluations_total{
              accepted="false"
            }[5m]) > 0.1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High policy denial rate detected"
            description: "Policy {{ $labels.policy_name }} is denying more than 0.1 req/s"

        # Alert if PolicyServer is down
        - alert: KubewardenPolicyServerDown
          expr: |
            up{job="kubewarden-policy-server"} == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Kubewarden PolicyServer is down"

        # Alert on slow policy evaluations
        - alert: KubewardenSlowPolicyEvaluation
          expr: |
            histogram_quantile(0.99,
              rate(kubewarden_policy_evaluation_duration_seconds_bucket[5m])
            ) > 1.0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Slow Kubewarden policy evaluations"
```

## Creating a Policy Compliance Dashboard Script

```bash
#!/bin/bash
# kubewarden-compliance-report.sh

echo "=== Kubewarden Policy Compliance Report ==="
echo "Date: $(date)"
echo ""

echo "--- Active Policies ---"
kubectl get clusteradmissionpolicies \
  -o custom-columns=\
'NAME:.metadata.name,\
MODULE:.spec.module,\
MODE:.spec.mode,\
MUTATING:.spec.mutating,\
ACTIVE:.status.conditions[?(@.type=="PolicyActive")].status'

echo ""
echo "--- Recent Policy Violations (last 1 hour) ---"
kubectl get events -A \
  --field-selector reason=PolicyViolation \
  --sort-by='.lastTimestamp' \
  | awk -v cutoff="$(date -d '1 hour ago' '+%Y-%m-%dT%H:%M:%S')" \
    '$1 > cutoff {print}'

echo ""
echo "--- Policies in Monitor Mode (review needed) ---"
kubectl get clusteradmissionpolicies \
  -o jsonpath='{range .items[?(@.spec.mode=="monitor")]}{.metadata.name}: MONITOR MODE - review violations before enforcing{"\n"}{end}'
```

## Conclusion

Comprehensive monitoring of Kubewarden policy decisions transforms your admission control from a black box into a transparent, auditable security layer. By combining Prometheus metrics for trends, OpenTelemetry traces for debugging, Kubernetes events for real-time visibility, and automated alerts for anomalies, you maintain full visibility into how your policies are performing. This observability foundation is especially important during policy rollout phases when you need to understand the impact of new policies before enabling enforcement mode.
