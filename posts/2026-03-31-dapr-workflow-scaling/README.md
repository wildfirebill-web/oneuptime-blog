# How to Scale Dapr Workflow Execution

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflow, Scaling, Performance, Kubernetes, Concurrency, Throughput

Description: Learn how to scale Dapr Workflow execution horizontally in Kubernetes, tune concurrency settings, and optimize throughput for high-volume workflow workloads.

---

## How Dapr Workflow Scales

Dapr Workflow uses a distributed orchestration runtime based on the Durable Task Framework. Workflow state is stored in an external state store (typically Redis or a database). Multiple app replicas can process different workflow instances simultaneously, providing horizontal scalability.

## Understanding the Dapr Scheduler

The Dapr Scheduler service distributes work items (activity tasks, workflow orchestration tasks) across available workers. Each sidecar pulls work from the scheduler. Scaling your app pods scales your workflow processing capacity.

## Horizontal Pod Autoscaling in Kubernetes

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: workflow-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: workflow-worker
  minReplicas: 2
  maxReplicas: 20
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
        averageUtilization: 80
```

## Deployment Configuration for Workflow Workers

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workflow-worker
spec:
  replicas: 5
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "workflow-worker"
        dapr.io/app-port: "6000"
        dapr.io/config: "workflowConfig"
    spec:
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
```

## Tuning Workflow Concurrency in Python SDK

Limit concurrent workflow and activity executions per worker instance:

```python
import dapr.ext.workflow as wf

# Configure worker concurrency
wf_runtime = wf.WorkflowRuntime(
    host="localhost",
    port=50001,
    # Maximum concurrent orchestrations per worker
    maximum_concurrent_workflow_instances=100,
    # Maximum concurrent activity executions per worker
    maximum_concurrent_activity_instances=200
)
```

## State Store Configuration for High Throughput

Use a performant state store for workflow state. Redis Cluster is recommended for high-volume workloads:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: workflowStateStore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-cluster:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-secret
      key: password
  - name: maxRetries
    value: "3"
  - name: maxRetryBackoff
    value: "5s"
  - name: poolSize
    value: "100"
```

## Scaling the Dapr Scheduler

In Kubernetes, scale the Dapr Scheduler for high-throughput deployments:

```bash
kubectl scale deployment dapr-scheduler --replicas=3 -n dapr-system
```

Or configure it in the Dapr Helm chart:

```yaml
# values.yaml
dapr_scheduler:
  replicaCount: 3
  resources:
    requests:
      cpu: "1000m"
      memory: "1Gi"
    limits:
      cpu: "4000m"
      memory: "4Gi"
```

## Monitoring Workflow Throughput

Use Prometheus metrics exposed by Dapr to track workflow throughput:

```bash
# Scrape Dapr metrics
curl http://localhost:9090/metrics | grep dapr_workflow
```

Key metrics to monitor:

```
dapr_workflow_activity_execution_time_bucket - activity execution latency
dapr_workflow_orchestration_execution_time_bucket - orchestration latency
dapr_workflow_activity_execution_total - total activities executed
```

## Partitioning Workflows for Scale

For extremely high volumes, partition workflows by a key (tenant, region) so work distributes evenly:

```python
# Use a partition key as part of the instance ID
instance_id = f"order-{region}-{order_id}"
```

## Limiting Parallelism in Fan-Out

When a workflow fans out to thousands of parallel activities, limit concurrency to avoid overwhelming downstream services:

```python
def process_batch_workflow(ctx: wf.DaprWorkflowContext, batch: dict):
    records = batch["records"]
    chunk_size = 50  # process 50 at a time

    for i in range(0, len(records), chunk_size):
        chunk = records[i:i+chunk_size]
        tasks = [ctx.call_activity(process_record, input=r) for r in chunk]
        yield wf.when_all(tasks)
```

## Summary

Dapr Workflow scales horizontally by adding more app replicas, with the Dapr Scheduler distributing work across all workers. Tune concurrency settings per worker to balance throughput with resource usage, use a Redis Cluster state store for high-volume scenarios, and partition workflow instances to distribute load evenly across the cluster.
