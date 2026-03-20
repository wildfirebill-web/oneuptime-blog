# How to Configure Kubernetes Cluster Policies in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Cluster Policies, Security, Governance

Description: Learn how to configure Kubernetes cluster policies in Portainer to enforce resource quotas, deployment standards, and security constraints.

## What Are Cluster Policies?

Cluster policies in Portainer define constraints and defaults applied to all workloads deployed through Portainer in a Kubernetes environment. They help enforce organizational standards without requiring developers to remember policy details.

## Accessing Cluster Policies

1. Select your Kubernetes environment in Portainer.
2. Go to **Cluster > Setup**.
3. Find the **Cluster policies** section.

## Available Policy Options

### Allow Privileged Containers

When disabled, Portainer prevents users from deploying containers with `privileged: true` in their security context:

```yaml
# This spec would be blocked when privileged containers are disabled
spec:
  containers:
    - name: app
      securityContext:
        privileged: true  # Blocked by policy
```

### Allow Bind Mounts

Controls whether users can mount host directories into containers. Disabling this prevents potential host filesystem exposure:

```yaml
# This volume type would be blocked
volumes:
  - name: host-path
    hostPath:
      path: /etc  # Blocked when bind mounts are disabled
```

### Allow Host Network Mode

Prevents containers from using `hostNetwork: true`, which would give them direct access to the host network stack.

## Namespace-Level Resource Quotas

Enforce resource limits at the namespace level to prevent resource exhaustion:

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: my-namespace
spec:
  hard:
    # Limit total CPU and memory in the namespace
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    # Limit number of resources
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
```

```bash
# Apply the quota
kubectl apply -f resource-quota.yaml

# Check quota usage in a namespace
kubectl describe resourcequota -n my-namespace
```

## LimitRange for Default Pod Resources

Require every pod to have resource requests and limits by setting defaults:

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: my-namespace
spec:
  limits:
    - type: Container
      # Default limits applied if not specified by the user
      default:
        cpu: 200m
        memory: 256Mi
      # Default requests if not specified
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      # Hard caps per container
      max:
        cpu: "2"
        memory: 2Gi
```

```bash
kubectl apply -f limit-range.yaml
```

## Enforcing Policies via Portainer

In Portainer's cluster setup:
- Toggle policies on/off for each Kubernetes environment.
- Portainer validates deployments against these policies at deploy time and blocks non-compliant configurations.

## Conclusion

Cluster policies in Portainer provide a first line of governance for Kubernetes workloads. Combine Portainer policies with native Kubernetes constructs (ResourceQuota, LimitRange) and admission controllers (OPA Gatekeeper) for a comprehensive policy enforcement strategy.
