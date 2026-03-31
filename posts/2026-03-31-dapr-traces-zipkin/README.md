# How to Send Dapr Traces to Zipkin

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Zipkin, Distributed Tracing, Observability, Microservice, Monitoring

Description: Configure Dapr to export trace data to Zipkin for visualizing service dependencies and diagnosing latency issues across microservices.

---

## Overview

Zipkin is a lightweight distributed tracing system that helps gather timing data needed to troubleshoot latency problems in microservice architectures. Dapr has first-class support for Zipkin through its built-in tracing configuration, making setup straightforward. This guide covers deploying Zipkin and configuring Dapr to send traces to it.

## Deploying Zipkin

### On Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zipkin
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zipkin
  template:
    metadata:
      labels:
        app: zipkin
    spec:
      containers:
        - name: zipkin
          image: openzipkin/zipkin:3
          ports:
            - containerPort: 9411
          env:
            - name: STORAGE_TYPE
              value: "mem"
---
apiVersion: v1
kind: Service
metadata:
  name: zipkin
  namespace: monitoring
spec:
  selector:
    app: zipkin
  ports:
    - port: 9411
      targetPort: 9411
```

```bash
kubectl apply -f zipkin.yaml
```

### Locally with Docker

```bash
docker run -d --name zipkin -p 9411:9411 openzipkin/zipkin:3
```

## Configuring Dapr to Send Traces to Zipkin

### Kubernetes Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: zipkin-tracing
  namespace: default
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: "http://zipkin.monitoring.svc.cluster.local:9411/api/v2/spans"
```

```bash
kubectl apply -f zipkin-config.yaml
```

### Self-Hosted Configuration

Create `~/.dapr/config.yaml`:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprConfig
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: "http://localhost:9411/api/v2/spans"
```

## Applying to Your Service

Add the `dapr.io/config` annotation to your deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "checkout-service"
        dapr.io/app-port: "8080"
        dapr.io/config: "zipkin-tracing"
```

For self-hosted, pass the config flag:

```bash
dapr run --app-id checkout-service \
  --app-port 8080 \
  --config ~/.dapr/config.yaml \
  ./checkout-service
```

## Generating Traces

Make a service invocation call between two Dapr services:

```bash
# Service A calls Service B
curl -X POST http://localhost:3500/v1.0/invoke/payment-service/method/charge \
  -H "Content-Type: application/json" \
  -d '{"amount": 59.99, "orderId": "ORD-101"}'
```

## Viewing Traces in Zipkin UI

```bash
kubectl port-forward -n monitoring svc/zipkin 9411:9411
```

Open `http://localhost:9411` and:
1. Click "Run Query" to find recent traces
2. Click a trace to see the waterfall view
3. Each span shows service name, operation, duration, and tags

## Zipkin API for Programmatic Access

```bash
# Get recent traces for a service
curl "http://localhost:9411/api/v2/traces?serviceName=checkout-service&limit=5"

# Look up a specific trace
curl "http://localhost:9411/api/v2/trace/4bf92f3577b34da6a3ce929d0e0e4736"

# Get all service names
curl "http://localhost:9411/api/v2/services"
```

## Persistent Storage with Elasticsearch

For production, use Elasticsearch storage:

```bash
docker run -d --name zipkin \
  -e STORAGE_TYPE=elasticsearch \
  -e ES_HOSTS=http://elasticsearch:9200 \
  -p 9411:9411 \
  openzipkin/zipkin:3
```

## Summary

Dapr has built-in Zipkin support through the `zipkin.endpointAddress` tracing configuration. Deploy Zipkin (use Elasticsearch for production persistence), create a Dapr `Configuration` resource pointing at Zipkin's API endpoint, and add the `dapr.io/config` annotation to your deployments. Dapr automatically generates and exports spans for all service invocations, pub/sub operations, and state management calls.
