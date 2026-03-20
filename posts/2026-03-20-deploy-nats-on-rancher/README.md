# How to Deploy NATS on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, NATS, Kubernetes, Messaging, Helm, JetStream

Description: Deploy a NATS messaging cluster on Rancher with JetStream persistence enabled, clustering configured, and connection testing validated.

## Introduction

NATS is an ultra-lightweight, high-performance cloud-native messaging system. It supports publish-subscribe, request-reply, and JetStream (persistent messaging). NATS is ideal for microservices communication due to its minimal resource footprint.

## Prerequisites

- Rancher cluster with `kubectl` and `helm` configured
- StorageClass available for JetStream persistence

## Step 1: Add NATS Helm Repository

```bash
helm repo add nats https://nats-io.github.io/k8s/helm/charts/
helm repo update
```

## Step 2: Create Values File

```yaml
# nats-values.yaml
cluster:
  enabled: true
  replicas: 3    # Three-node cluster

nats:
  jetstream:
    enabled: true    # Enable persistent messaging
    fileStorage:
      enabled: true
      size: 20Gi
      storageDirectory: /data/jetstream
      storageClassName: longhorn

  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "500m"

exporter:
  enabled: true    # Prometheus metrics exporter
```

## Step 3: Deploy NATS

```bash
kubectl create namespace messaging

helm install nats nats/nats \
  --namespace messaging \
  --values nats-values.yaml
```

## Step 4: Verify Cluster

```bash
# Check pods
kubectl get pods -n messaging -l app.kubernetes.io/name=nats

# Check NATS cluster status
kubectl exec -it nats-0 -n messaging -- \
  nats-server --version
```

## Step 5: Test Publish/Subscribe

Use the NATS CLI to test messaging.

```bash
# Install the NATS CLI (from inside a pod or locally)
kubectl exec -it nats-0 -n messaging -- sh

# Subscribe to a subject
nats sub test.subject --server nats://localhost:4222 &

# Publish a message
nats pub test.subject "Hello from NATS!" --server nats://localhost:4222
```

## Step 6: Create a JetStream Stream

JetStream adds persistence and replay capabilities to NATS subjects.

```bash
# Create a durable stream
nats stream add EVENTS \
  --subjects "events.*" \
  --storage file \
  --replicas 3 \
  --max-msgs=-1 \
  --max-bytes=-1 \
  --max-age=24h \
  --server nats://nats.messaging.svc.cluster.local:4222
```

## Step 7: Connect Applications

```yaml
# Application environment variable for NATS connection
env:
  - name: NATS_URL
    value: "nats://nats.messaging.svc.cluster.local:4222"
```

## Conclusion

NATS is running on Rancher with clustering and JetStream persistence. Its lightweight nature (under 100MB memory for a basic cluster) makes it suitable for edge deployments and high-frequency microservice communication. The built-in Prometheus exporter enables metric collection without additional configuration.
