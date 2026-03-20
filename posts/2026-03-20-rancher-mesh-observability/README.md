# How to Configure Service Mesh Observability in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Service Mesh, Observability, Prometheus, Jaeger, Kiali

Description: Configure comprehensive observability for your service mesh in Rancher using Prometheus, Grafana, Jaeger, and Kiali to gain full visibility into service-to-service communication.

## Introduction

Service mesh observability provides deep insights into microservice communication, including request rates, error rates, latency distributions, and distributed traces. This guide covers setting up a complete observability stack for Istio-based service meshes in Rancher, including metrics with Prometheus/Grafana, distributed tracing with Jaeger, and topology visualization with Kiali.

## Prerequisites

- Rancher with Istio installed
- Rancher Monitoring (Prometheus/Grafana) stack deployed
- kubectl with cluster-admin access
- Helm 3.x

## Step 1: Install Rancher Monitoring

Install the Rancher Monitoring stack from the Apps catalog:

```bash
# Install Rancher Monitoring via Helm
helm repo add rancher-charts https://charts.rancher.io
helm install rancher-monitoring rancher-charts/rancher-monitoring \
  --namespace cattle-monitoring-system \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=standard \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

## Step 2: Enable Istio Metrics Collection

Configure Istio to expose metrics to Prometheus:

```yaml
# istio-telemetry.yaml - Enable Istio metrics and tracing
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: default-telemetry
  namespace: istio-system
spec:
  metrics:
    - providers:
        - name: prometheus
  # Configure distributed tracing
  tracing:
    - providers:
        - name: jaeger
      # Sample 100% of traces in dev, reduce in production
      randomSamplingPercentage: 1.0
```

## Step 3: Configure ServiceMonitor for Istio

```yaml
# istio-service-monitor.yaml - Prometheus scrape config for Istio
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-component-monitor
  namespace: istio-system
  labels:
    monitoring: istio-components
    release: rancher-monitoring
spec:
  jobLabel: istio
  targetLabels:
    - app
  selector:
    matchExpressions:
      - key: istio
        operator: In
        values:
          - mixer
          - pilot
          - galley
          - citadel
          - sidecar-injector
  namespaceSelector:
    matchNames:
      - istio-system
  endpoints:
    - port: http-monitoring
      interval: 15s
---
# Service monitor for Envoy sidecars
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: envoy-stats-monitor
  namespace: istio-system
  labels:
    monitoring: istio-proxies
    release: rancher-monitoring
spec:
  selector:
    matchExpressions:
      - key: istio-prometheus-ignore
        operator: DoesNotExist
  namespaceSelector:
    any: true
  jobLabel: envoy-stats
  endpoints:
    - path: /stats/prometheus
      targetPort: 15090
      interval: 15s
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_container_port_name]
          action: keep
          regex: ".*-envoy-prom"
```

## Step 4: Install Kiali for Service Topology

```bash
# Install Kiali from the Rancher Apps catalog or via Helm
helm repo add kiali https://kiali.org/helm-charts
helm install kiali-operator kiali/kiali-operator \
  --namespace kiali-operator \
  --create-namespace
```

```yaml
# kiali-cr.yaml - Kiali instance configuration
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  name: kiali
  namespace: istio-system
spec:
  deployment:
    accessible_namespaces:
      - "**"  # Monitor all namespaces
  external_services:
    prometheus:
      url: http://rancher-monitoring-prometheus.cattle-monitoring-system.svc.cluster.local:9090
    grafana:
      enabled: true
      url: http://rancher-monitoring-grafana.cattle-monitoring-system.svc.cluster.local:80
    tracing:
      enabled: true
      in_cluster_url: http://jaeger-query.istio-system.svc.cluster.local:16685
      use_grpc: true
  auth:
    strategy: anonymous  # Use 'openshift' or 'token' for production
```

## Step 5: Install Jaeger for Distributed Tracing

```bash
# Install Jaeger Operator
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm install jaeger-operator jaegertracing/jaeger-operator \
  --namespace observability \
  --create-namespace
```

```yaml
# jaeger-instance.yaml - Jaeger AllInOne for development
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch.observability.svc.cluster.local:9200
  query:
    serviceType: ClusterIP
  collector:
    maxReplicas: 5
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
```

## Step 6: Configure Istio to Use Jaeger

```yaml
# istio-operator.yaml - Configure Istio tracing
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-control-plane
  namespace: istio-system
spec:
  meshConfig:
    # Send traces to Jaeger collector
    defaultConfig:
      tracing:
        zipkin:
          address: jaeger-collector.observability.svc.cluster.local:9411
    # Sample 1% in production (increase for debugging)
    accessLogFile: /dev/stdout
    enableTracing: true
```

## Step 7: Create Custom Grafana Dashboards

Import Istio dashboards:

```bash
# Download and apply Istio Grafana dashboards
DASHBOARDS=(
  "7630"   # Istio Workload Dashboard
  "7636"   # Istio Service Dashboard
  "7645"   # Istio Control Plane Dashboard
  "7639"   # Istio Mesh Dashboard
)

for ID in "${DASHBOARDS[@]}"; do
  echo "Importing Grafana dashboard $ID"
  curl -s "https://grafana.com/api/dashboards/$ID/revisions/latest/download" | \
    kubectl create configmap "grafana-dashboard-$ID" \
    --from-file="dashboard-${ID}.json=/dev/stdin" \
    --namespace=cattle-monitoring-system
done
```

## Step 8: Create Prometheus Alerting Rules

```yaml
# mesh-alerts.yaml - Alert rules for service mesh health
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: istio-mesh-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: istio.rules
      rules:
        # Alert if service error rate exceeds 5%
        - alert: IstioHighErrorRate
          expr: |
            sum(rate(istio_requests_total{
              reporter="destination",
              response_code!~"5.*"
            }[5m])) by (destination_service) /
            sum(rate(istio_requests_total{
              reporter="destination"
            }[5m])) by (destination_service) < 0.95
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High error rate for {{ $labels.destination_service }}"
            description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.destination_service }}"

        # Alert if P99 latency exceeds 1 second
        - alert: IstioHighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (destination_service, le)
            ) > 1000
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High P99 latency for {{ $labels.destination_service }}"
            description: "P99 latency is {{ $value }}ms for {{ $labels.destination_service }}"
```

## Step 9: Access Observability Tools

```bash
# Port-forward Kiali
kubectl port-forward -n istio-system svc/kiali 20001:20001

# Port-forward Jaeger
kubectl port-forward -n observability svc/jaeger-query 16686:16686

# Port-forward Grafana
kubectl port-forward -n cattle-monitoring-system svc/rancher-monitoring-grafana 3000:80
```

## Conclusion

A complete service mesh observability stack transforms a black box microservices architecture into a fully transparent system. The combination of Prometheus metrics, Grafana dashboards, Jaeger distributed tracing, and Kiali topology visualization gives you everything needed to understand, debug, and optimize inter-service communication. In Rancher environments, the built-in monitoring stack provides an excellent foundation that integrates naturally with Istio's telemetry capabilities.
