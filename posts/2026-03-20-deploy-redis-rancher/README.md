# How to Deploy Redis on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rancher, Kubernetes, Helm, StatefulSets, Caching, Database, SUSE Rancher

Description: Learn how to deploy a production-ready Redis cluster on a Rancher-managed Kubernetes cluster using the Bitnami Helm chart with persistent storage, authentication, and Sentinel for high availability.

---

Redis on Kubernetes with Rancher provides a highly available, in-memory data store for caching, session management, and pub/sub messaging. The Bitnami chart supports both standalone and Redis Sentinel deployments.

---

## Step 1: Add the Bitnami Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## Step 2: Create a Values File

```yaml
# redis-values.yaml

architecture: replication   # Deploy with 1 master + 2 replicas

auth:
  enabled: true
  password: ""              # Auto-generated; retrieve with kubectl get secret

master:
  persistence:
    enabled: true
    storageClass: longhorn
    size: 10Gi
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi

replica:
  replicaCount: 2
  persistence:
    enabled: true
    storageClass: longhorn
    size: 10Gi
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi

# Redis Sentinel for automatic failover
sentinel:
  enabled: true
  masterSet: mymaster
  quorum: 2

# Enable Prometheus metrics
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
```

---

## Step 3: Deploy Redis

```bash
# Create namespace
kubectl create namespace redis

# Install Redis
helm install redis bitnami/redis \
  --namespace redis \
  --values redis-values.yaml \
  --wait

# Verify pods
kubectl get pods -n redis
```

---

## Step 4: Get the Redis Password

```bash
# Retrieve the auto-generated password
kubectl get secret redis \
  -n redis \
  -o jsonpath='{.data.redis-password}' | base64 -d
```

---

## Step 5: Connect to Redis

```bash
# Connect to the Redis master
kubectl exec -it redis-master-0 -n redis -- \
  redis-cli -a $(
    kubectl get secret redis -n redis \
    -o jsonpath='{.data.redis-password}' | base64 -d
  )

# Test basic operations
127.0.0.1:6379> SET mykey "hello"
127.0.0.1:6379> GET mykey
127.0.0.1:6379> INFO replication
```

---

## Step 6: Check Sentinel Status

```bash
# Connect to Sentinel and verify master discovery
kubectl exec -it redis-node-0 -n redis -- \
  redis-cli -p 26379 SENTINEL masters

# Expected output includes:
# name: mymaster
# ip: 10.x.x.x
# port: 6379
# flags: master
```

---

## Step 7: Connect from an Application

When Redis Sentinel is enabled, applications connect through Sentinel for automatic failover:

```bash
# Create a connection secret for applications
kubectl create secret generic redis-connection \
  --namespace default \
  --from-literal=host="redis.redis.svc.cluster.local" \
  --from-literal=port="6379" \
  --from-literal=sentinel-hosts="redis.redis.svc.cluster.local:26379" \
  --from-literal=sentinel-master="mymaster" \
  --from-literal=password="$(
    kubectl get secret redis -n redis \
    -o jsonpath='{.data.redis-password}' | base64 -d
  )"
```

Use the secret in your application deployment:

```yaml
env:
  - name: REDIS_HOST
    valueFrom:
      secretKeyRef:
        name: redis-connection
        key: host
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef:
        name: redis-connection
        key: password
```

---

## Step 8: Monitor Redis Performance

```promql
# Redis memory usage
redis_memory_used_bytes

# Redis commands per second
rate(redis_commands_processed_total[5m])

# Redis connected clients
redis_connected_clients

# Keyspace hit rate
rate(redis_keyspace_hits_total[5m]) /
  (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
```

---

## Persistence Configuration for Caching vs. Data Store

```yaml
# For pure caching (no persistence needed - faster, lower storage cost)
master:
  persistence:
    enabled: false
replica:
  persistence:
    enabled: false

# For session storage or queue (AOF persistence for durability)
master:
  extraFlags:
    - "--appendonly yes"
    - "--appendfsync everysec"
```

---

## Best Practices

- Enable Redis Sentinel (`sentinel.enabled: true`) for production deployments - it provides automatic failover without requiring application-level cluster awareness.
- Use a dedicated namespace for Redis and restrict network access using NetworkPolicy.
- For caching workloads, disable persistence to improve write performance and reduce storage costs; for session storage or queues, enable AOF persistence.
