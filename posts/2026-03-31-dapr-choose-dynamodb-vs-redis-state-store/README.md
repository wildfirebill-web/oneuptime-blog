# How to Choose Between AWS DynamoDB and Redis for Dapr State Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS, DynamoDB, Redis, State Store, Comparison

Description: Compare AWS DynamoDB and Redis as Dapr state stores, covering durability, latency, cost, and consistency to help you choose the right backend for your workload.

---

## Overview

AWS DynamoDB and Redis are two of the most popular Dapr state store backends for cloud-native microservices. They serve different purposes: Redis is an in-memory store optimized for speed, while DynamoDB is a fully managed durable NoSQL database. This guide helps you decide which one fits your requirements.

## Head-to-Head Comparison

| Feature | AWS DynamoDB | Redis (ElastiCache) |
|---------|-------------|---------------------|
| Durability | Fully durable (SSD) | Memory (optional AOF/RDB) |
| Latency | Single-digit ms | Sub-millisecond |
| Max value size | 400KB | 512MB |
| Auto-scaling | Automatic | Manual or Cluster |
| TTL support | Native | Native |
| Strong consistency | Optional | Eventual (Cluster), Strong (single) |
| Global replication | DynamoDB Global Tables | Redis Global Datastore |
| Cost model | Pay per operation | Instance-based |
| Cold start | No | No (data in memory) |

## When to Choose AWS DynamoDB

DynamoDB is the better choice when:
- State must survive restarts, memory failures, or infrastructure events
- You have large state payloads (up to 400KB per item)
- You need predictable costs at variable throughput using PAY_PER_REQUEST billing
- Compliance or audit requirements mandate durable storage

DynamoDB Dapr configuration:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: dynamodb-statestore
spec:
  type: state.aws.dynamodb
  version: v1
  metadata:
  - name: table
    value: "DaprState"
  - name: region
    value: "us-east-1"
  - name: ttlAttributeName
    value: "TTL"
```

## When to Choose Redis

Redis is the better choice when:
- Sub-millisecond latency is required (gaming, real-time pricing)
- State is ephemeral or session-based and can be rebuilt if lost
- You need pub/sub and state in the same infrastructure component
- High write throughput (>100K ops/sec) is needed

Redis Dapr configuration:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: redis-statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-cluster.default.svc.cluster.local:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-secret
      key: password
  - name: enableTLS
    value: "true"
```

## Latency Benchmark Example

```bash
# Test DynamoDB latency through Dapr
time curl http://localhost:3500/v1.0/state/dynamodb-statestore/test-key
# Typical: 5-10ms

# Test Redis latency through Dapr
time curl http://localhost:3500/v1.0/state/redis-statestore/test-key
# Typical: 0.5-2ms
```

## Hybrid Pattern: Redis + DynamoDB

For the best of both worlds, use Redis as a hot cache in front of DynamoDB:

```javascript
async function getState(key) {
  // Try Redis first (fast, ephemeral)
  let value = await client.state.get("redis-statestore", key);
  if (value) return value;

  // Fall back to DynamoDB (durable, slower)
  value = await client.state.get("dynamodb-statestore", key);
  if (value) {
    // Warm the Redis cache
    await client.state.save("redis-statestore", [{ key, value }]);
  }
  return value;
}
```

## Summary

Choose DynamoDB when durability, large item support, and predictable costs matter most. Choose Redis when sub-millisecond latency and high write throughput are the priority. For mission-critical applications, the hybrid cache-aside pattern combining both backends provides the best balance of performance and durability.
