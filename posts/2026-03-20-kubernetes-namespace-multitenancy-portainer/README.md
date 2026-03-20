# How to Implement Namespace-Based Multi-Tenancy in Portainer for Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Namespaces, Multi-Tenant, RBAC, Access Control

Description: Configure Kubernetes namespace-based multi-tenancy through Portainer, assigning teams to specific namespaces with RBAC policies and resource quotas for complete tenant isolation.

## Introduction

Kubernetes namespaces provide the primary isolation boundary for multi-tenancy. Combined with Portainer's Kubernetes RBAC integration, you can give each team or tenant access to only their namespace through the Portainer UI - without them needing kubectl access or knowledge of other namespaces. This guide covers creating namespaced environments, applying ResourceQuotas, and configuring Portainer team access for Kubernetes.

## Step 1: Connect Kubernetes Cluster to Portainer

```bash
# Install Portainer Agent in the Kubernetes cluster

kubectl apply -n portainer -f \
  https://downloads.portainer.io/ce2-19/portainer-agent-k8s-lb.yaml

# Get the agent's LoadBalancer IP
kubectl get svc -n portainer portainer-agent

# Register the cluster in Portainer
# Portainer UI: Environments > Add Environment > Kubernetes
# URL: https://AGENT_LB_IP:9001
```

## Step 2: Create Namespaces per Tenant

```bash
# Create namespaces for each team
kubectl create namespace team-alpha
kubectl create namespace team-beta
kubectl create namespace team-gamma

# Label namespaces for Portainer management
kubectl label namespace team-alpha portainer.team=alpha tenant=alpha-corp
kubectl label namespace team-beta portainer.team=beta tenant=beta-inc
kubectl label namespace team-gamma portainer.team=gamma tenant=gamma-ltd

# Verify
kubectl get namespaces --show-labels
```

## Step 3: Apply Resource Quotas per Namespace

```yaml
# team-alpha-quota.yaml - Resource limits for Team Alpha's namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    # Compute resources
    requests.cpu: "4"          # Total CPU requests allowed
    requests.memory: 8Gi       # Total memory requests
    limits.cpu: "8"            # Total CPU limits
    limits.memory: 16Gi        # Total memory limits

    # Object count limits
    pods: "20"                 # Max 20 pods
    services: "10"             # Max 10 services
    persistentvolumeclaims: "5" # Max 5 PVCs
    secrets: "20"              # Max 20 secrets
    configmaps: "20"           # Max 20 configmaps
    deployments.apps: "10"     # Max 10 deployments
```

```bash
kubectl apply -f team-alpha-quota.yaml

# Apply LimitRange for default container limits
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: team-alpha-limits
  namespace: team-alpha
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"       # Default CPU limit per container
      memory: "512Mi"   # Default memory limit per container
    defaultRequest:
      cpu: "100m"       # Default CPU request
      memory: "128Mi"   # Default memory request
    max:
      cpu: "2"          # Max CPU limit per container
      memory: "4Gi"     # Max memory limit per container
EOF
```

## Step 4: Create RBAC for Tenant Users

```yaml
# team-alpha-rbac.yaml - RBAC for Team Alpha
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-alpha-sa
  namespace: team-alpha

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-alpha-role
  namespace: team-alpha
rules:
  # Allow managing deployments, pods, services in their namespace
  - apiGroups: ["", "apps", "extensions"]
    resources:
      ["deployments", "pods", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Allow viewing logs
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]
    verbs: ["get", "list"]
  # Deny access to other namespaces (this role only applies to team-alpha)

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-binding
  namespace: team-alpha
subjects:
  - kind: ServiceAccount
    name: team-alpha-sa
    namespace: team-alpha
roleRef:
  kind: Role
  name: team-alpha-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f team-alpha-rbac.yaml

# Create kubeconfig for Team Alpha (scoped to their namespace only)
kubectl get secret \
  $(kubectl get serviceaccount team-alpha-sa -n team-alpha \
    -o jsonpath='{.secrets[0].name}') \
  -n team-alpha -o jsonpath='{.data.token}' | base64 -d
```

## Step 5: Configure Portainer Namespace Access per Team

```bash
PORTAINER_URL="https://portainer.example.com"
ADMIN_TOKEN="admin_token"
K8S_ENV_ID=2  # Kubernetes environment ID

# Get team IDs
ALPHA_TEAM=$(curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  "$PORTAINER_URL/api/teams" | \
  jq -r '.[] | select(.Name == "Alpha Corp") | .Id')

# Restrict Team Alpha to team-alpha namespace only
curl -s -X PUT \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/kubernetes/$K8S_ENV_ID/namespaces/team-alpha/access" \
  -d "{
    \"AuthorizedTeams\": [$ALPHA_TEAM]
  }"

# Team Alpha users can now only deploy to team-alpha namespace
# They cannot see or modify team-beta or team-gamma namespaces
```

## Step 6: Verify Namespace Isolation

```bash
# Check quota usage for each namespace
kubectl describe quota -n team-alpha
kubectl describe quota -n team-beta

# Verify cross-namespace isolation
# As a team-alpha user, attempt to access team-beta namespace
kubectl get pods -n team-beta --as=system:serviceaccount:team-alpha:team-alpha-sa
# Error: forbidden: User "system:serviceaccount:team-alpha:team-alpha-sa"
# cannot list resource "pods" in API group "" in the namespace "team-beta"

# Check resource usage
kubectl top pods -n team-alpha
kubectl top pods -n team-beta

# List namespaces with their quota status
kubectl get resourcequota --all-namespaces
```

## Conclusion

Kubernetes namespace-based multi-tenancy combines several components: namespaces provide logical isolation, ResourceQuotas enforce compute limits, LimitRanges set per-container defaults, RBAC restricts which users can act in each namespace, and NetworkPolicies (not covered here) provide network isolation. Portainer's Kubernetes integration surfaces these namespaces in the UI and maps Portainer teams to specific namespaces, providing a friendly interface for teams who don't need to learn kubectl. This architecture scales well - adding a new tenant means creating a namespace, applying quota and RBAC manifests, and configuring Portainer access for the team.
