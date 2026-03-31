# How to Use Redis Cluster with Dapr State Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Redis, Redis Cluster, State Store, High Availability

Description: Learn how to configure Dapr to use Redis Cluster as a state store, enabling horizontal scaling and automatic sharding for high-throughput microservice state management.

---

## Overview

Redis Cluster distributes data across multiple nodes using hash slots, enabling horizontal scaling beyond what a single Redis instance can handle. When your Dapr state store needs to serve millions of keys or handle high write throughput, Redis Cluster is the right solution. This guide covers deploying Redis Cluster and configuring Dapr to use it.

## Deploying Redis Cluster on Kubernetes

Use the Bitnami Redis Cluster chart:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install redis-cluster bitnami/redis-cluster \
  --set cluster.nodes=6 \
  --set cluster.replicas=1 \
  --set password=clusterpassword \
  --set persistence.size=10Gi \
  --namespace redis \
  --create-namespace
```

Verify the cluster is healthy:

```bash
kubectl exec -n redis redis-cluster-0 -- \
  redis-cli -a clusterpassword cluster info | grep cluster_state
```

Expected output: `cluster_state:ok`

## Configuring Dapr for Redis Cluster

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: redis-statestore
  namespace: default
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-cluster.redis.svc.cluster.local:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-cluster-secret
      key: password
  - name: enableTLS
    value: "false"
  - name: maxRetries
    value: "3"
  - name: maxRetryBackoff
    value: "2s"
  - name: poolSize
    value: "10"
  - name: redisType
    value: "cluster"
```

The key setting is `redisType: cluster`, which enables Dapr to use the Redis Cluster protocol for key routing.

Create the secret:

```bash
kubectl create secret generic redis-cluster-secret \
  --from-literal=password=clusterpassword
```

Apply the component:

```bash
kubectl apply -f redis-cluster-statestore.yaml
```

## Using the Cluster State Store

State operations work identically to a single-node Redis setup:

```javascript
import { DaprClient } from "@dapr/dapr";

const client = new DaprClient();

// Save state - Dapr handles key routing to the correct cluster shard
await client.state.save("redis-statestore", [
  { key: "product-catalog-v3", value: { items: 15000, updated: "2026-03-31" } }
]);

const catalog = await client.state.get("redis-statestore", "product-catalog-v3");
console.log("Catalog:", catalog);
```

## Multi-Key Operations with Cluster

Redis Cluster restricts multi-key operations to keys in the same hash slot. Use hash tags to group related keys:

```bash
# Keys with the same hash tag {user:42} will be in the same slot
curl -X POST http://localhost:3500/v1.0/state/redis-statestore \
  -H "Content-Type: application/json" \
  -d '[
    {"key": "{user:42}:profile", "value": {"name": "Alice"}},
    {"key": "{user:42}:preferences", "value": {"theme": "dark"}}
  ]'
```

## Monitoring Cluster Health

```bash
# Check slot distribution across nodes
kubectl exec -n redis redis-cluster-0 -- \
  redis-cli -a clusterpassword cluster nodes

# Check memory usage per shard
for i in 0 1 2 3 4 5; do
  echo "Node $i:"
  kubectl exec -n redis redis-cluster-$i -- \
    redis-cli -a clusterpassword info memory | grep used_memory_human
done
```

## Summary

Redis Cluster with Dapr provides horizontal scalability for state management at high volume. Setting `redisType: cluster` in the Dapr component is the critical configuration step, while using hash tags ensures that related keys are co-located on the same shard for efficient multi-key transactions.
