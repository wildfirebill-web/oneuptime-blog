# How to Deploy Jaeger on Rancher for Distributed Tracing - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Jaeger, Distributed Tracing, Observability

Description: Deploy Jaeger distributed tracing on Rancher-managed clusters to visualize request flows across microservices and diagnose latency issues.

## Introduction

Jaeger is an open-source distributed tracing system that helps developers monitor and troubleshoot microservices. By instrumenting your services with OpenTelemetry or Jaeger client libraries, you get end-to-end visibility into request flows across your Kubernetes workloads. This guide covers deploying Jaeger on Rancher with production-grade storage backends.

## Prerequisites

- Rancher-managed Kubernetes cluster
- Helm 3.x installed
- kubectl with cluster-admin access
- Elasticsearch or Cassandra for production storage

## Step 1: Install the Jaeger Operator

```bash
# Add Jaeger Helm repository

helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

# Install the Jaeger Operator
helm install jaeger-operator jaegertracing/jaeger-operator \
  --namespace observability \
  --create-namespace \
  --set rbac.clusterRole=true \
  --wait

# Verify operator is running
kubectl get pods -n observability
```

## Step 2: Deploy Jaeger All-in-One (Development)

```yaml
# jaeger-allinone.yaml - Simple all-in-one Jaeger for development
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-dev
  namespace: observability
spec:
  strategy: allInOne
  allInOne:
    image: jaegertracing/all-in-one:latest
    options:
      log-level: info
  storage:
    type: memory  # In-memory storage (development only)
    options:
      memory:
        max-traces: 100000
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - jaeger.dev.example.com
```

## Step 3: Deploy Production Jaeger with Elasticsearch

```bash
# First, deploy Elasticsearch (or use existing)
helm install elasticsearch bitnami/elasticsearch \
  --namespace observability \
  --set master.replicaCount=3 \
  --set data.replicaCount=3 \
  --wait
```

```yaml
# jaeger-production.yaml - Production Jaeger with Elasticsearch
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-prod
  namespace: observability
spec:
  strategy: production  # Separate collector and query components

  collector:
    maxReplicas: 5
    resources:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 512Mi
    options:
      # Collector settings
      collector.num-workers: 50
      collector.queue-size: 2000

  query:
    replicas: 2
    resources:
      requests:
        memory: 256Mi
        cpu: 100m
    options:
      query.base-path: /jaeger
    metricsStorage:
      type: prometheus

  storage:
    type: elasticsearch
    options:
      es.server-urls: http://elasticsearch.observability.svc.cluster.local:9200
      es.index-prefix: jaeger
      es.num-shards: 5
      es.num-replicas: 1
      es.create-index-templates: true
    esIndexCleaner:
      enabled: true
      numberOfDays: 30
      schedule: "55 23 * * *"

  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - jaeger.example.com
    tls:
      - secretName: jaeger-tls
        hosts:
          - jaeger.example.com
```

## Step 4: Configure Application Instrumentation

### Using OpenTelemetry SDK (Recommended)

```python
# Python application with OpenTelemetry instrumentation
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Configure tracer
provider = TracerProvider()
# Send traces to OpenTelemetry Collector which forwards to Jaeger
exporter = OTLPSpanExporter(
    endpoint="http://otel-collector.observability.svc.cluster.local:4317"
)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

# Instrument an HTTP handler
@app.route('/api/orders')
def get_orders():
    with tracer.start_as_current_span("get-orders") as span:
        span.set_attribute("http.method", "GET")
        span.set_attribute("http.url", request.url)

        orders = fetch_orders_from_db()
        span.set_attribute("orders.count", len(orders))
        return jsonify(orders)
```

### Configure Jaeger Agent as DaemonSet

```yaml
# jaeger-agent-daemonset.yaml - Jaeger agent on every node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: jaeger-agent
  namespace: observability
spec:
  selector:
    matchLabels:
      app: jaeger-agent
  template:
    metadata:
      labels:
        app: jaeger-agent
    spec:
      containers:
        - name: jaeger-agent
          image: jaegertracing/jaeger-agent:latest
          args:
            - --reporter.grpc.host-port=jaeger-prod-collector.observability.svc.cluster.local:14250
            - --log-level=info
          ports:
            - containerPort: 6831  # UDP for compact Thrift protocol
              protocol: UDP
              hostPort: 6831
            - containerPort: 6832  # UDP for binary Thrift protocol
              protocol: UDP
              hostPort: 6832
            - containerPort: 5778  # HTTP for configs
              hostPort: 5778
```

## Step 5: Configure Sampling Strategy

```yaml
# sampling-config.yaml - Adaptive sampling configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-sampling-config
  namespace: observability
data:
  sampling: |
    {
      "service_strategies": [
        {
          "service": "order-service",
          "type": "probabilistic",
          "param": 0.1  # Sample 10% of order service traces
        },
        {
          "service": "payment-service",
          "type": "probabilistic",
          "param": 1.0  # Sample 100% of payment traces
        }
      ],
      "default_strategy": {
        "type": "probabilistic",
        "param": 0.01  # Default: 1% sampling
      }
    }
```

## Step 6: Access Jaeger UI

```bash
# Port forward to Jaeger query UI
kubectl port-forward -n observability svc/jaeger-prod-query 16686:16686

# Access at: http://localhost:16686

# Or via Ingress if configured
# https://jaeger.example.com
```

## Step 7: Query Traces via API

```bash
# Search for traces via API
curl "http://localhost:16686/api/traces?service=order-service&limit=20&lookback=1h" | jq '.data[0]'

# Get trace by ID
curl "http://localhost:16686/api/traces/abc123def456" | jq '.'

# Get services list
curl "http://localhost:16686/api/services" | jq '.'
```

## Troubleshooting

```bash
# Check Jaeger collector logs
kubectl logs -n observability deployment/jaeger-prod-collector --tail=100

# Check if traces are being received
kubectl logs -n observability deployment/jaeger-prod-collector | grep "Span accepted"

# Check Elasticsearch index
curl http://elasticsearch.observability.svc.cluster.local:9200/_cat/indices/jaeger* | sort
```

## Conclusion

Jaeger provides comprehensive distributed tracing for microservices on Rancher. Start with all-in-one deployment for development, then migrate to production mode with Elasticsearch for persistent, queryable trace storage. Instrument your applications with OpenTelemetry for vendor-neutral tracing that can be routed to Jaeger, Zipkin, or other backends. The combination of Jaeger with Rancher's monitoring stack gives you complete observability across your service mesh.
