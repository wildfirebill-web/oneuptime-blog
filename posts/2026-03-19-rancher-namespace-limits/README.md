# How to Set Resource Limits on Namespaces in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Namespace, Resource Quota

Description: Learn how to configure resource limits on individual namespaces in Rancher to control resource consumption and prevent workload interference.

While project-level quotas control total resource consumption across a group of namespaces, namespace-level limits give you finer control over individual namespaces. This guide covers how to set, customize, and manage resource limits on namespaces within Rancher projects.

## Prerequisites

- Rancher v2.7+ with cluster owner or project owner access
- A project with at least one namespace
- Understanding of Kubernetes resource requests and limits

## Understanding Namespace Resource Limits

Namespace resource limits in Rancher come from two sources:

1. **Project namespace default limits**: When a project has quotas configured with namespace defaults, every new namespace in that project automatically receives a ResourceQuota.
2. **Per-namespace overrides**: You can customize the limits for individual namespaces within the project's total allocation.

Additionally, you can set **LimitRange** resources to control default and maximum resource allocations per container or pod within a namespace.

## Step 1: View Current Namespace Limits

Check what limits are currently applied to a namespace:

```bash
# View ResourceQuota in a namespace

kubectl get resourcequota -n <namespace-name> -o yaml

# View LimitRange in a namespace
kubectl get limitrange -n <namespace-name> -o yaml

# Summary view
kubectl describe resourcequota -n <namespace-name>
kubectl describe limitrange -n <namespace-name>
```

## Step 2: Set Namespace Limits via the Rancher UI

1. Navigate to your cluster in Rancher.
2. Go to **Cluster > Projects/Namespaces**.
3. Find the namespace you want to configure.
4. Click the three-dot menu and select **Edit**.
5. Under **Resource Limits**, configure the limits:
   - **CPU Reservation**: Total CPU requests allowed
   - **CPU Limit**: Total CPU limits allowed
   - **Memory Reservation**: Total memory requests allowed
   - **Memory Limit**: Total memory limits allowed
   - **Pods**: Maximum number of pods
6. Click **Save**.

## Step 3: Set Namespace Limits via kubectl

Create a ResourceQuota for the namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: api-production
spec:
  hard:
    pods: "50"
    requests.cpu: "8"
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
    configmaps: "50"
    persistentvolumeclaims: "10"
    secrets: "50"
    services: "20"
    services.loadbalancers: "2"
    services.nodeports: "5"
```

```bash
kubectl apply -f namespace-quota.yaml
```

## Step 4: Set Container-Level Defaults with LimitRange

LimitRange objects set default resource requests and limits for containers that do not specify their own:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-defaults
  namespace: api-production
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
        cpu: "2"
        memory: "4Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: Pod
      max:
        cpu: "4"
        memory: "8Gi"
    - type: PersistentVolumeClaim
      max:
        storage: "50Gi"
      min:
        storage: "1Gi"
```

```bash
kubectl apply -f limitrange.yaml
```

This configuration:

- Sets default container limits to 500m CPU and 512Mi memory
- Sets default container requests to 100m CPU and 128Mi memory
- Caps any single container at 2 CPU and 4Gi memory
- Caps any single pod at 4 CPU and 8Gi memory
- Limits PVC sizes between 1Gi and 50Gi

## Step 5: Configure Default Container Resource Limits in Rancher

Rancher projects can set default container resource limits that apply to all namespaces:

1. Go to **Cluster > Projects/Namespaces**.
2. Click the three-dot menu on the project.
3. Select **Edit Config**.
4. Scroll to **Container Default Resource Limit**.
5. Configure:
   - **CPU Reservation**: Default CPU request per container
   - **CPU Limit**: Default CPU limit per container
   - **Memory Reservation**: Default memory request per container
   - **Memory Limit**: Default memory limit per container
6. Click **Save**.

Rancher creates LimitRange objects in each namespace to enforce these defaults.

## Step 6: Override Namespace Limits Within a Project

When one namespace needs more resources than the default:

```bash
# Check the project's total quota and current usage
kubectl get projects.management.cattle.io <project-id> -n <cluster-id> -o json | \
  jq '.spec.resourceQuota'

# Check how much is allocated to other namespaces
kubectl get resourcequota --all-namespaces -o json | \
  jq -r '.items[] | select(.metadata.namespace | startswith("api-")) | "\(.metadata.namespace): CPU=\(.spec.hard["requests.cpu"] // "none"), Memory=\(.spec.hard["requests.memory"] // "none")"'
```

Then adjust the specific namespace's quota:

```bash
kubectl patch resourcequota namespace-quota -n api-production -p '{
  "spec": {
    "hard": {
      "requests.cpu": "16",
      "requests.memory": "32Gi",
      "pods": "100"
    }
  }
}'
```

Make sure the total across all namespaces does not exceed the project limit.

## Step 7: Set Limits for Different Workload Types

Create multiple LimitRanges for different pod categories using labels and admission webhooks. A simpler approach is to set per-namespace limits based on workload type:

**For high-traffic API namespaces:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: api-quota
  namespace: api-production
spec:
  hard:
    pods: "100"
    requests.cpu: "16"
    requests.memory: "32Gi"
    limits.cpu: "32"
    limits.memory: "64Gi"
```

**For batch processing namespaces:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: batch-quota
  namespace: batch-processing
spec:
  hard:
    pods: "20"
    requests.cpu: "32"
    requests.memory: "64Gi"
    limits.cpu: "64"
    limits.memory: "128Gi"
```

Batch namespaces typically need fewer pods but more resources per pod.

## Step 8: Monitor Namespace Resource Usage

Track how namespaces use their allocated resources:

```bash
# Current usage vs limits
kubectl get resourcequota -n api-production -o json | \
  jq '.items[] | .status | {
    used: .used,
    hard: .hard
  }'

# Percentage usage
kubectl get resourcequota -n api-production -o json | \
  jq -r '.items[].status | to_entries | group_by(.key | split(".")[0]) | .[] | .[0].key as $type |
  if ($type == "used" or $type == "hard") then empty else . end'
```

For a visual overview, use Rancher's monitoring:

1. Navigate to the cluster.
2. Go to **Monitoring > Dashboards**.
3. Look for namespace-level resource usage dashboards.

## Step 9: Handle Resource Limit Issues

Common issues and solutions:

**Pods stuck in Pending due to quota:**

```bash
# Check if quota is exhausted
kubectl describe resourcequota -n <namespace>

# Find pods consuming the most resources
kubectl top pods -n <namespace> --sort-by=cpu
kubectl top pods -n <namespace> --sort-by=memory
```

**Pods rejected due to LimitRange:**

```bash
# Check the LimitRange constraints
kubectl describe limitrange -n <namespace>

# The error message will indicate which constraint was violated
# Adjust the pod spec or the LimitRange accordingly
```

**Quota not applied to new namespace:**

```bash
# Verify the namespace is in the correct project
kubectl get namespace <namespace> -o jsonpath='{.metadata.annotations.field\.cattle\.io/projectId}'

# Re-apply the namespace to the project if needed
kubectl annotate namespace <namespace> \
  field.cattle.io/projectId="c-m-xxxxx:p-xxxxx" --overwrite
```

## Step 10: Automate Namespace Limit Configuration

Use a script to apply consistent limits across namespaces:

```bash
#!/bin/bash
# apply-namespace-limits.sh

NAMESPACE=$1
TIER=${2:-standard}  # standard, high, or batch

case $TIER in
  standard)
    CPU_REQ="4" ; CPU_LIM="8" ; MEM_REQ="8Gi" ; MEM_LIM="16Gi" ; PODS="50"
    ;;
  high)
    CPU_REQ="16" ; CPU_LIM="32" ; MEM_REQ="32Gi" ; MEM_LIM="64Gi" ; PODS="100"
    ;;
  batch)
    CPU_REQ="32" ; CPU_LIM="64" ; MEM_REQ="64Gi" ; MEM_LIM="128Gi" ; PODS="20"
    ;;
esac

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ${TIER}-quota
  namespace: ${NAMESPACE}
spec:
  hard:
    pods: "${PODS}"
    requests.cpu: "${CPU_REQ}"
    requests.memory: "${MEM_REQ}"
    limits.cpu: "${CPU_LIM}"
    limits.memory: "${MEM_LIM}"
EOF

echo "Applied $TIER limits to namespace $NAMESPACE"
```

Usage:

```bash
./apply-namespace-limits.sh api-production high
./apply-namespace-limits.sh data-processing batch
./apply-namespace-limits.sh frontend-staging standard
```

## Best Practices

- **Always set both requests and limits**: Resource quotas that only set limits without requests can lead to unpredictable scheduling.
- **Use LimitRange for defaults**: Ensure every container has resource requests and limits even if developers forget to set them.
- **Right-size over time**: Start with estimates and adjust based on actual monitoring data.
- **Set minimum resource requirements**: Use LimitRange `min` to prevent containers from requesting too few resources.
- **Cap individual containers**: Use LimitRange `max` to prevent any single container from consuming too many resources.
- **Monitor continuously**: Set up alerts for when namespaces approach their limits.

## Conclusion

Setting resource limits on namespaces in Rancher provides precise control over resource consumption at the namespace level. By combining ResourceQuotas for total namespace limits and LimitRanges for per-container defaults and bounds, you create a predictable resource environment. Start with project-level defaults, customize for specific namespaces, and monitor continuously to keep allocations aligned with actual needs.
