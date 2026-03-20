# How to Deploy NATS on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, NATS, Message Queue, Cloud Native Messaging

Description: Deploy NATS messaging system on Rancher-managed clusters for lightweight, high-performance pub/sub messaging and streaming with JetStream.

## Introduction

NATS is a simple, secure, high-performance open-source messaging system designed for cloud-native applications. It supports pub/sub, request-reply, and queue group patterns. NATS JetStream adds persistent messaging, stream replay, and consumer groups. This guide covers deploying a NATS cluster on Rancher with JetStream enabled.

## Prerequisites

- Rancher-managed Kubernetes cluster
- Helm 3.x installed
- kubectl access
- A StorageClass for JetStream persistence

## Step 1: Deploy NATS with Helm

```bash
# Add NATS Helm repository
helm repo add nats https://nats-io.github.io/k8s/helm/charts/
helm repo update

# Check available versions
helm search repo nats/nats
```

```yaml
# nats-values.yaml - NATS cluster configuration
config:
  cluster:
    enabled: true
    replicas: 3
    name: "rancher-nats"

  jetstream:
    enabled: true
    fileStore:
      enabled: true
      pvc:
        enabled: true
        storageClassName: standard
        size: 20Gi

  auth:
    enabled: true
    resolver:
      type: full
      operator: myoperator
      systemAccount: SYS
      store:
        dir: /etc/nats/jwt
        size: 10Mi

  tls:
    enabled: true
    secret:
      name: nats-server-tls

  websocket:
    enabled: true
    port: 8080

container:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

natsbox:
  enabled: true

reloader:
  enabled: true

promExporter:
  enabled: true
  port: 7777
  podMonitor:
    enabled: true
    namespace: cattle-monitoring-system
    labels:
      release: rancher-monitoring
```

```bash
# Deploy NATS
helm install nats nats/nats \
  --namespace messaging \
  --create-namespace \
  --values nats-values.yaml \
  --wait

# Verify deployment
kubectl get pods -n messaging -l app.kubernetes.io/name=nats
```

## Step 2: Verify NATS Cluster

```bash
# Access nats-box for testing
kubectl exec -n messaging -it deployment/nats-box -- \
  /bin/sh

# Inside nats-box
nats server check --server nats://nats.messaging.svc.cluster.local:4222

# View server info
nats server info --server nats://nats.messaging.svc.cluster.local:4222
```

## Step 3: Configure JetStream Streams

```bash
# Create a stream
kubectl exec -n messaging deployment/nats-box -- \
  nats stream create ORDERS \
  --server nats://nats.messaging.svc.cluster.local:4222 \
  --subjects "orders.*" \
  --storage file \
  --replicas 3 \
  --retention limits \
  --max-age 7d \
  --max-msgs 10000000 \
  --discard old \
  --no-allow-rollup-hdrs

# Create a durable consumer
kubectl exec -n messaging deployment/nats-box -- \
  nats consumer create ORDERS order-processor \
  --server nats://nats.messaging.svc.cluster.local:4222 \
  --filter "orders.new" \
  --pull \
  --deliver all \
  --max-deliver 5 \
  --ack explicit \
  --wait 30s
```

## Step 4: Create NATS Accounts and Users

```bash
# Install NSC for NATS account management
curl -L https://raw.githubusercontent.com/nats-io/nsc/main/install.sh | sh

# Create an operator
nsc add operator myoperator

# Create accounts
nsc add account --name APP
nsc add account --name SYS

# Create users
nsc add user --account APP --name appuser

# Export credentials
nsc generate creds -n appuser -a APP > appuser.creds

# Create Kubernetes secret for credentials
kubectl create secret generic nats-creds \
  --from-file=appuser.creds=appuser.creds \
  --namespace=production
```

## Step 5: Configure Applications to Use NATS

```yaml
# app-deployment.yaml - Application using NATS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:v1.0
          env:
            - name: NATS_URL
              value: "nats://nats.messaging.svc.cluster.local:4222"
            - name: NATS_CREDS
              value: "/etc/nats/creds/appuser.creds"
          volumeMounts:
            - name: nats-creds
              mountPath: /etc/nats/creds
              readOnly: true
      volumes:
        - name: nats-creds
          secret:
            secretName: nats-creds
```

## Step 6: Publish and Subscribe Test

```bash
# Start a subscriber
kubectl exec -n messaging deployment/nats-box -- \
  nats sub "orders.*" \
  --server nats://nats.messaging.svc.cluster.local:4222 &

# Publish a test message
kubectl exec -n messaging deployment/nats-box -- \
  nats pub "orders.new" \
  '{"order_id": "123", "item": "widget", "quantity": 5}' \
  --server nats://nats.messaging.svc.cluster.local:4222

# Test JetStream publish
kubectl exec -n messaging deployment/nats-box -- \
  nats stream add --defaults EVENTS \
  --subjects "events.>" \
  --server nats://nats.messaging.svc.cluster.local:4222

kubectl exec -n messaging deployment/nats-box -- \
  nats pub --count 10 "events.test" \
  "Test message {{.Count}}" \
  --server nats://nats.messaging.svc.cluster.local:4222
```

## Step 7: Monitor NATS

```bash
# View NATS server metrics
kubectl port-forward -n messaging svc/nats 8222:8222 &

# Check server metrics via HTTP monitoring
curl http://localhost:8222/varz | jq .
curl http://localhost:8222/connz | jq .
curl http://localhost:8222/routez | jq .
curl http://localhost:8222/jsz | jq .  # JetStream info
```

## Step 8: Configure NATS Leaf Nodes for Multi-Cluster

```yaml
# nats-leaf-values.yaml - Leaf node for remote cluster
config:
  leafnodes:
    enabled: true
    remotes:
      - url: "nats://nats.messaging.central-cluster.svc.cluster.local:7422"
        credentials: /etc/nats/leaf.creds
  jetstream:
    enabled: true
```

## Troubleshooting

```bash
# Check NATS server logs
kubectl logs -n messaging nats-0 --tail=100

# Check cluster routes
kubectl exec -n messaging deployment/nats-box -- \
  nats server report routes \
  --server nats://nats.messaging.svc.cluster.local:4222

# Check JetStream info
kubectl exec -n messaging deployment/nats-box -- \
  nats server report jetstream \
  --server nats://nats.messaging.svc.cluster.local:4222

# Check stream health
kubectl exec -n messaging deployment/nats-box -- \
  nats stream info ORDERS \
  --server nats://nats.messaging.svc.cluster.local:4222
```

## Conclusion

NATS on Rancher provides an extremely lightweight, high-performance messaging solution ideal for cloud-native microservices. NATS Core offers simple pub/sub with minimal overhead, while JetStream adds durable streaming capabilities for scenarios requiring guaranteed delivery and message replay. The NATS server's small footprint and operational simplicity make it an excellent choice for microservice communication in resource-constrained environments.
