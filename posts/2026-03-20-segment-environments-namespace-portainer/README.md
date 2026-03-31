# How to Segment Environments with Namespace Access in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Namespace, Multi-Tenancy, Access Control

Description: Learn how to segment Kubernetes environments by namespace in Portainer to isolate teams and enforce workload separation.

## Namespace-Based Isolation in Portainer

Portainer's namespace isolation mode gives users access to only their assigned namespaces. This creates logical tenant boundaries within a single Kubernetes cluster without running multiple clusters.

## Configuring Namespace Isolation Mode

1. Select your Kubernetes environment in Portainer.
2. Go to **Cluster > Setup**.
3. Under **Security**, find **Namespace access mode**.
4. Select:
   - **Shared access**: Users see all authorized namespaces.
   - **Isolated access**: Users only see their own namespace(s).
5. Click **Save changes**.

## Creating Namespace Assignments

Assign teams to specific namespaces:

1. Go to **Namespaces**.
2. Click on a namespace.
3. Click **Manage access**.
4. Add teams or users with appropriate roles.

## Example: Three-Team Namespace Segmentation

```bash
# Create namespaces for three teams

kubectl create namespace team-frontend
kubectl create namespace team-backend
kubectl create namespace team-data

# Label namespaces for clear ownership
kubectl label namespace team-frontend team=frontend owner=frontend-team
kubectl label namespace team-backend team=backend owner=backend-team
kubectl label namespace team-data team=data owner=data-team
```

In Portainer, assign each namespace to its corresponding team.

## Network Isolation Between Namespaces

Namespace separation in Portainer controls visibility, but network traffic can still flow between namespaces. Use NetworkPolicies for true network isolation:

```yaml
# Deny all ingress and egress by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-frontend
spec:
  podSelector: {}   # Applies to all pods
  policyTypes:
    - Ingress
    - Egress
---
# Allow frontend to communicate with backend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-to-backend
  namespace: team-frontend
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              team: backend   # Only allow traffic to backend namespace
      ports:
        - port: 8080
```

Resource Quota Per Team Namespace

Combine namespace segmentation with resource quotas:

```yaml
# team-frontend-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: frontend-quota
  namespace: team-frontend
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "30"
    services.loadbalancers: "1"  # Only 1 LoadBalancer per team
```

## RBAC for Namespace Segmentation

```yaml
# Restrict a team to only their namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-team-access
  namespace: team-frontend
subjects:
  - kind: Group
    name: frontend-team  # Mapped from Portainer team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit              # Can deploy and manage workloads
  apiGroup: rbac.authorization.k8s.io
```

## Verifying Isolation

```bash
# Verify that team-frontend cannot see team-backend resources
kubectl auth can-i get pods \
  --namespace=team-backend \
  --as=frontend-user@company.com
# Expected: no

kubectl auth can-i get pods \
  --namespace=team-frontend \
  --as=frontend-user@company.com
# Expected: yes
```

## Conclusion

Namespace-based segmentation in Portainer creates clean team boundaries within a shared Kubernetes cluster. Combine Portainer's namespace access control with Kubernetes RBAC and NetworkPolicies for a comprehensive multi-tenancy solution.
