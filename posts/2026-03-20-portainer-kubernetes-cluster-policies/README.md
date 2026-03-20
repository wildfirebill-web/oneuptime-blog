# How to Configure Kubernetes Cluster Policies in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Policies, Resource Management, DevOps

Description: Learn how to configure Kubernetes cluster policies in Portainer including resource quotas, limit ranges, and deployment restrictions.

## Introduction

Portainer provides cluster-level policy configuration for Kubernetes environments, allowing administrators to enforce resource limits, control deployments, and standardize workload configurations. This guide covers the available policy settings and how to configure them.

## Prerequisites

- Portainer BE with Kubernetes environment
- Cluster-admin or namespace-admin access

## Step 1: Access Cluster Policy Settings

1. Select your Kubernetes environment in Portainer
2. Navigate to **Settings → Cluster**
3. Scroll to **Cluster policies** or **Deployment options**

## Step 2: Configure Deployment Restrictions

Portainer BE allows restricting what users can deploy:

```
Deployment options:
  [ ] Require resources on namespaces   — Force resource quotas before allowing deployments
  [x] Allow web editor                  — Users can write raw YAML
  [x] Allow Helm charts                 — Users can deploy Helm charts
  [x] Allow use of node ports           — Allow NodePort service creation
  [ ] Restrict privileged containers    — Block privileged container deployments
```

## Step 3: Configure Default Namespace Resource Quotas

Set default resource quotas for new namespaces:

```yaml
# Apply default resource quota via Portainer YAML editor
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-quota
  namespace: new-namespace
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "10"
    secrets: "20"
    configmaps: "20"
```

## Step 4: Configure LimitRanges

LimitRanges enforce per-container resource constraints:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    # Default limits for containers without specified limits
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 4Gi
      min:
        cpu: 50m
        memory: 64Mi

    # Limits for pods
    - type: Pod
      max:
        cpu: "8"
        memory: 8Gi
```

## Step 5: Configure Pod Disruption Budgets

Ensure high availability during updates and maintenance:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
  namespace: production
spec:
  minAvailable: 2    # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: web-frontend
```

Or using maxUnavailable:

```yaml
spec:
  maxUnavailable: 1   # Allow at most 1 pod to be unavailable
  selector:
    matchLabels:
      app: api
```

## Step 6: Configure Network Policies as Default

Deploy default network policies for new namespaces:

```yaml
# Template for new namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

## Step 7: Enforce Resource Policies with OPA/Gatekeeper

For complex policy enforcement, deploy OPA Gatekeeper:

```bash
# Install Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml
```

Example constraint to require resource limits on all containers:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - kube-public
  parameters:
    limits:
      - cpu
      - memory
    requests:
      - cpu
      - memory
```

## Step 8: Portainer BE Cluster Policies

Configure in Portainer BE under **Settings → Cluster**:

### Resource Over-commit

```
Allow resource over-commit:  [ ] disabled
# When disabled: sum of resource requests cannot exceed cluster capacity
# Prevents scheduling failures due to over-commitment
```

### Default Namespace Isolation

```
Restrict default namespace: [x] enabled
# Prevents deployments to the 'default' namespace
# Forces use of dedicated namespaces
```

## Step 9: Monitor Policy Compliance

```bash
# Check for pods without resource limits
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.limits == null) |
  "\(.metadata.namespace)/\(.metadata.name)"'

# Check for pods running as root
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].securityContext.runAsNonRoot != true) |
  "\(.metadata.namespace)/\(.metadata.name)"'
```

## Conclusion

Kubernetes cluster policies in Portainer help administrators maintain consistency and safety across deployments. Start with resource quotas and limit ranges to prevent resource exhaustion, add Pod Disruption Budgets for availability guarantees, and use OPA Gatekeeper for complex policy requirements. Portainer BE's cluster settings complement these Kubernetes-native policies with UI-level controls.
