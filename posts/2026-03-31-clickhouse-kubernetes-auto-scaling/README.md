# How to Set Up ClickHouse Auto-Scaling on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Kubernetes, Auto-Scaling, HPA, VPA, KEDA, Resource Management

Description: Learn how to configure horizontal and vertical auto-scaling for ClickHouse on Kubernetes using HPA, VPA, and KEDA to handle variable analytical workloads.

---

Auto-scaling ClickHouse on Kubernetes requires careful consideration - ClickHouse is not a stateless service. Horizontal scaling adds new shard/replica nodes, while vertical scaling adjusts CPU and memory per pod. This guide covers both approaches using Kubernetes-native tools.

## Why ClickHouse Scaling Is Different

ClickHouse shards data - adding nodes does not automatically rebalance existing data. Auto-scaling is most effective for:
- **Vertical scaling**: Adjusting memory/CPU on existing nodes
- **Read replica scaling**: Adding read-only replicas without resharding
- **Compute-storage separation**: Scaling query nodes in ClickHouse Cloud architecture

## Vertical Pod Autoscaler (VPA)

VPA adjusts CPU and memory requests/limits based on actual usage:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: clickhouse-vpa
  namespace: clickhouse
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: clickhouse
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: clickhouse
        minAllowed:
          cpu: "2"
          memory: "4Gi"
        maxAllowed:
          cpu: "16"
          memory: "64Gi"
```

## Horizontal Pod Autoscaler for Read Replicas

For ClickHouse setups with dedicated read replicas:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: clickhouse-read-hpa
  namespace: clickhouse
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: clickhouse-read
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
```

## KEDA for Query-Driven Scaling

Use KEDA to scale based on ClickHouse query queue depth:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: clickhouse-scaler
spec:
  scaleTargetRef:
    name: clickhouse-workers
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: clickhouse_query_queue_size
        threshold: "10"
        query: clickhouse_query_thread_pool_jobs_count{pool="Query"}
```

## Resource Requests Configuration

Always set resource requests and limits for ClickHouse pods:

```yaml
resources:
  requests:
    cpu: "4"
    memory: "8Gi"
  limits:
    cpu: "8"
    memory: "16Gi"
```

## Monitoring for Scaling Decisions

```sql
SELECT
    toStartOfMinute(event_time) AS minute,
    avg(memory_usage) / 1e9 AS avg_memory_gb,
    max(memory_usage) / 1e9 AS max_memory_gb
FROM system.query_log
WHERE event_time > now() - INTERVAL 1 HOUR
  AND type = 'QueryFinish'
GROUP BY minute
ORDER BY minute;
```

## Summary

Auto-scaling ClickHouse on Kubernetes works best with VPA for right-sizing individual nodes, HPA for read replica pools, and KEDA for workload-driven scaling. Avoid indiscriminate horizontal scaling of shards, as it requires manual data rebalancing. Monitor memory and CPU in `system.query_log` to inform scaling thresholds.
