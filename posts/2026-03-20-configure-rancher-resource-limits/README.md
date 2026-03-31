# How to Configure Rancher Resource Limits

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Resource Limit, Kubernetes, LimitRange, ResourceQuota, Cost Optimization

Description: Configure resource limits and quotas in Rancher at cluster, project, and namespace levels to ensure fair resource distribution and prevent resource contention.

## Introduction

Resource management in Rancher operates at three levels: cluster (hardware capacity), project (team allocation), and namespace (workload isolation). Properly configured limits prevent a single workload from starving others and help Kubernetes make better scheduling decisions.

## Step 1: Configure Project Resource Quotas in Rancher

Rancher Projects provide a higher-level abstraction above namespaces:

1. Navigate to your cluster in the Rancher UI
2. Go to **Projects/Namespaces > Create Project**
3. Enable **Project Resource Quotas** and set limits:

```yaml
# Via Rancher API

apiVersion: management.cattle.io/v3
kind: Project
metadata:
  name: production-team
spec:
  resourceQuota:
    limit:
      limitsCpu: "20000m"      # 20 CPU cores for the entire project
      limitsMemory: "40Gi"     # 40GB memory for the entire project
      requestsStorage: "500Gi"
  namespaceDefaultResourceQuota:
    limit:
      limitsCpu: "4000m"       # Default 4 CPUs per namespace
      limitsMemory: "8Gi"
```

## Step 2: Configure LimitRanges

LimitRanges set per-container defaults for namespaces without explicit limits:

```yaml
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: Pod
      max:
        cpu: "8"
        memory: "16Gi"
    - type: PersistentVolumeClaim
      max:
        storage: "100Gi"    # Prevent oversized PVC requests
```

## Step 3: Set Resource Requirements in Deployments

All production workloads must have explicit resource requests and limits:

```yaml
# deployment.yaml
spec:
  containers:
    - name: api
      image: myapi:v1.0
      resources:
        requests:
          cpu: "200m"       # Guaranteed CPU allocation (used for scheduling)
          memory: "256Mi"   # Guaranteed memory allocation
        limits:
          cpu: "1"          # Maximum CPU (container is throttled above this)
          memory: "1Gi"     # Maximum memory (container is OOM-killed above this)
```

## Step 4: Configure QoS Classes

Kubernetes assigns QoS classes based on resource configuration:

```yaml
# Guaranteed QoS (best protection from eviction)
# Set requests == limits for all containers
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# Burstable QoS (normal workloads)
# requests < limits
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

## Step 5: Monitor Resource Usage vs Limits

```bash
# View resource usage across a namespace
kubectl top pods -n production --sort-by=memory

# Check if quotas are being hit
kubectl describe resourcequota -n production

# Find pods approaching memory limits
kubectl get pods -n production -o json | \
  jq -r '.items[] | select(.status.phase=="Running") | .metadata.name'
```

## Conclusion

Resource limits in Rancher protect cluster stability by preventing workloads from consuming disproportionate resources. Set `requests` based on actual usage (measure with `kubectl top`) and `limits` at 2-3x the request value to allow bursting. Always configure Guaranteed QoS for stateful workloads like databases.
