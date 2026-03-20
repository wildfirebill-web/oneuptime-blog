# How to Configure Per-Team Resource Quotas in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Resource Quotas, Teams, Multi-Tenancy, Business Edition, Kubernetes

Description: Learn how to configure per-team resource quotas in Portainer Business Edition for Docker and Kubernetes environments to prevent resource overconsumption.

---

Resource quotas prevent a single team from monopolizing CPU, memory, or storage on a shared infrastructure. Portainer Business Edition supports quotas for Kubernetes namespaces and Docker environments.

## Portainer Business Edition Resource Quotas

Portainer BE integrates with Kubernetes ResourceQuotas and Docker resource limits to enforce per-team boundaries.

### Kubernetes Namespace Quotas

In Kubernetes environments, assign each team to a namespace and apply a ResourceQuota:

```yaml
# Apply a ResourceQuota for Team A's namespace

apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "4"        # Total CPU requests capped at 4 cores
    requests.memory: 8Gi     # Total memory requests capped at 8 GB
    limits.cpu: "8"          # Total CPU limits capped at 8 cores
    limits.memory: 16Gi      # Total memory limits capped at 16 GB
    pods: "20"               # Maximum 20 pods
    services: "10"           # Maximum 10 services
    persistentvolumeclaims: "5"  # Maximum 5 PVCs
```

Apply this via Portainer's Kubernetes resource management or kubectl:

```bash
kubectl apply -f team-a-quota.yaml
```

View the quota status in Portainer under **Kubernetes > Namespaces > team-a > Resource Quotas**.

### LimitRange for Default Container Limits

Prevent containers without resource specifications from using unlimited resources:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-limits
  namespace: team-a
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: 256Mi
      defaultRequest:
        cpu: "100m"
        memory: 64Mi
      max:
        cpu: "2"
        memory: 2Gi
```

This ensures every container in the namespace has limits, even if the developer didn't specify them.

## Docker Stack Resource Limits

For Docker environments, enforce limits at the stack level. Add a `resources` block to every service in tenant stacks:

```yaml
version: "3.8"

services:
  api:
    image: my-api:latest
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
```

## Portainer BE: Environment-Level Resource Quotas

Portainer Business Edition allows setting resource quotas directly in the environment settings:

1. Go to **Environments > [environment name] > Configuration**.
2. Under **Resource limits**, set CPU and memory quotas.
3. These apply as Kubernetes ResourceQuotas or Docker daemon constraints.

## Monitoring Resource Usage per Team

Track actual resource usage against quotas:

```bash
# Kubernetes: check quota usage
kubectl describe quota team-a-quota -n team-a

# Output shows current vs limit:
# Name: team-a-quota
# Namespace: team-a
# Resource          Used   Hard
# --------          ----   ----
# limits.cpu        2      8
# limits.memory     3Gi    16Gi
# pods              8      20
```

For Docker environments, check aggregate stats per team via the Portainer API or cAdvisor.

## Alerting on Quota Approach

Set up alerts when a team approaches their quota:

```bash
#!/bin/bash
# check-quota-usage.sh

kubectl get resourcequota -A -o json | jq -r '
  .items[] |
  .metadata.namespace as $ns |
  .status.hard as $hard |
  .status.used as $used |
  (.status.used["limits.memory"] // "0") as $used_mem |
  (.status.hard["limits.memory"] // "0") as $hard_mem |
  {namespace: $ns, used_memory: $used_mem, hard_memory: $hard_mem}
' | jq -r '. | "\(.namespace): \(.used_memory) / \(.hard_memory)"'
```

Use OneUptime to monitor these values and alert when usage exceeds 80% of the quota.
