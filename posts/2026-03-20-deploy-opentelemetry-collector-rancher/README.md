# How to Deploy OpenTelemetry Collector on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, OpenTelemetry, Observability, Traces, Metrics, Logs

Description: Deploy the OpenTelemetry Collector on Rancher as a DaemonSet and Gateway, configure pipelines to route telemetry to multiple backends.

## Introduction

The OpenTelemetry Collector is a vendor-agnostic agent that receives, processes, and exports telemetry data (traces, metrics, logs). Deploying it as a DaemonSet ensures every node collects telemetry, while a central Gateway collector aggregates and routes data to backends like Jaeger, Prometheus, and Loki.

## Collector Architecture

```mermaid
graph LR
    A[App Pods] -->|OTLP| B[DaemonSet Collector]
    B -->|OTLP| C[Gateway Collector]
    C -->|Traces| D[Jaeger]
    C -->|Metrics| E[Prometheus]
    C -->|Logs| F[Loki]
```

## Step 1: Add the OpenTelemetry Repository

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

## Step 2: Deploy the DaemonSet Collector

The DaemonSet collector runs on every node and collects node-level and pod-level telemetry.

```yaml
# otel-daemonset-values.yaml
mode: daemonset   # One pod per node

config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317    # Accept traces/metrics from apps
        http:
          endpoint: 0.0.0.0:4318
    kubeletstats:
      collection_interval: 20s
      auth_type: serviceAccount
      endpoint: "${env:K8S_NODE_NAME}:10250"

  processors:
    batch:
      timeout: 10s
    memory_limiter:
      limit_mib: 400

  exporters:
    otlp/gateway:
      endpoint: otel-gateway-collector.observability.svc.cluster.local:4317
      tls:
        insecure: true

  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [otlp/gateway]
      metrics:
        receivers: [otlp, kubeletstats]
        processors: [memory_limiter, batch]
        exporters: [otlp/gateway]
```

```bash
kubectl create namespace observability

helm install otel-daemonset open-telemetry/opentelemetry-collector \
  --namespace observability \
  --values otel-daemonset-values.yaml
```

## Step 3: Deploy the Gateway Collector

The Gateway aggregates all telemetry and routes to multiple backends.

```yaml
# otel-gateway-values.yaml
mode: deployment
replicaCount: 2

config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317

  exporters:
    jaeger:
      endpoint: jaeger-collector.observability.svc.cluster.local:14250
      tls:
        insecure: true
    prometheus:
      endpoint: 0.0.0.0:8889
    loki:
      endpoint: http://loki.observability.svc.cluster.local:3100/loki/api/v1/push

  service:
    pipelines:
      traces:
        receivers: [otlp]
        exporters: [jaeger]
      metrics:
        receivers: [otlp]
        exporters: [prometheus]
```

```bash
helm install otel-gateway open-telemetry/opentelemetry-collector \
  --namespace observability \
  --values otel-gateway-values.yaml
```

## Step 4: Instrument Applications

Configure applications to send telemetry to the DaemonSet collector using the node IP:

```yaml
# Application pod spec
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://$(HOST_IP):4317"
  - name: HOST_IP
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
  - name: OTEL_SERVICE_NAME
    value: "my-service"
```

## Conclusion

The OpenTelemetry Collector on Rancher provides a flexible telemetry pipeline that decouples application instrumentation from backend storage. Changing your trace or metrics backend requires only a collector configuration update—no application code changes needed.
