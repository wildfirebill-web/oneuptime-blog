# How to Segment Environments with Namespace Access in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Namespace, RBAC, Security

Description: Learn how to use Kubernetes namespaces with Portainer's access control to segment environments by team, ensuring each team only sees and manages their own workloads.

## Introduction

Namespace-based segmentation is the primary method for multi-tenancy in Kubernetes. Portainer's Business Edition extends this with its own RBAC layer, allowing you to map Portainer teams to Kubernetes namespaces so developers only see namespaces they have been granted access to. This guide covers complete namespace segmentation setup.

## Prerequisites

- Portainer BE with a Kubernetes environment
- Multiple teams already created in Portainer
- Admin access to Portainer
- kubectl access for namespace management

## Architecture: Team-to-Namespace Mapping

```text
Portainer Team: backend-engineers
    → Kubernetes Namespace: backend
    → Access level: Standard (can deploy)
    → Cannot see: frontend, operations, monitoring namespaces

Portainer Team: frontend-engineers
    → Kubernetes Namespace: frontend
    → Access level: Standard

Portainer Team: platform-devops
    → All namespaces
    → Access level: Environment Admin
```

## Step 1: Create Namespaces

```bash
# Create team-specific namespaces

kubectl create namespace backend
kubectl create namespace frontend
kubectl create namespace operations
kubectl create namespace monitoring
kubectl create namespace staging

# Apply labels for organization
kubectl label namespace backend team=backend env=production
kubectl label namespace frontend team=frontend env=production
kubectl label namespace staging env=staging
```

Or via Portainer UI:
1. Select your Kubernetes environment.
2. Go to **Namespaces** → **Add namespace**.
3. Enter the namespace name and labels.
4. Configure resource quotas if desired.

## Step 2: Assign Namespaces to Teams in Portainer

### Via Portainer UI

1. Go to **Namespaces** in your Kubernetes environment.
2. Click on a namespace (e.g., `backend`).
3. Scroll to **Access control**.
4. Click **Add access**.
5. Select the team: `backend-engineers`
6. Select the role: **Standard User**
7. Click **Apply changes**.

Repeat for each namespace-team pair.

### Via Portainer API

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"
ENDPOINT_ID=1

# Grant backend team access to backend namespace
# First, find the team ID
BACKEND_TEAM_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/teams" | \
  jq -r '.[] | select(.Name == "backend-engineers") | .Id')

# Set namespace access for the team
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/namespaces/backend/access" \
  -d "{
    \"teamAccessPolicies\": {
      \"${BACKEND_TEAM_ID}\": {\"RoleId\": 2}
    }
  }" | jq .
```

## Step 3: Enable Namespace Isolation

Portainer BE can enforce that users can only see namespaces they have access to:

1. Go to environment **Settings**.
2. Enable **Restrict default namespace**.
3. Enable **Namespace-based access control**.
4. Save.

After this:
- Backend team users see only the `backend` namespace
- Frontend team users see only the `frontend` namespace
- DevOps team (with environment admin) sees all namespaces

## Step 4: Kubernetes Network Policies for Hard Isolation

Add Kubernetes NetworkPolicies to prevent cross-namespace communication:

```yaml
# network-policy-isolate.yaml - Deny all cross-namespace traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: backend
spec:
  podSelector: {}  # Applies to all pods in this namespace
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}          # Allow from same namespace
  egress:
    - to:
        - podSelector: {}          # Allow to same namespace
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system  # Allow DNS
      ports:
        - protocol: UDP
          port: 53
```

```bash
# Apply to each team namespace
for NS in backend frontend operations; do
  sed "s/namespace: backend/namespace: $NS/" network-policy-isolate.yaml | \
    kubectl apply -f -
  echo "Applied isolation policy to namespace: $NS"
done
```

## Step 5: RBAC at the Kubernetes Level

Portainer's access control maps to Kubernetes RBAC. For additional control, create Kubernetes Roles:

```yaml
# role-developer.yaml - Developer role for a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: backend
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["pods", "deployments", "replicasets", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]
    verbs: ["get", "list", "create"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]  # Can read but not create/delete secrets
  - apiGroups: [""]
    resources: ["nodes", "namespaces"]
    verbs: []  # No access to cluster-level resources
```

## Step 6: View Access Configuration

```bash
# See what namespaces a user has access to (from their kubeconfig)
kubectl auth can-i list pods --all-namespaces  # Should be false for non-admin
kubectl auth can-i list pods -n backend         # Should be true for backend team

# Check RBAC bindings
kubectl get rolebindings -n backend
kubectl describe rolebinding -n backend

# Check namespace access from Portainer
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/namespaces" | \
  jq '.[] | {name: .Name, accessControl: .ResourceControl}'
```

## Step 7: Namespace Provisioning Script

```bash
#!/bin/bash
# provision-team-namespace.sh

TEAM_NAME=$1
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"
ENDPOINT_ID=1

# Create namespace
kubectl create namespace "$TEAM_NAME" --dry-run=client -o yaml | kubectl apply -f -

# Apply resource quota (4 CPU, 8GB memory for standard team)
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: $TEAM_NAME
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
EOF

# Find Portainer team ID
TEAM_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/teams" | \
  jq -r --arg n "$TEAM_NAME" '.[] | select(.Name == $n) | .Id')

# Grant namespace access
curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/namespaces/${TEAM_NAME}/access" \
  -d "{\"teamAccessPolicies\": {\"${TEAM_ID}\": {\"RoleId\": 2}}}" > /dev/null

echo "Namespace $TEAM_NAME provisioned for team $TEAM_NAME"
```

## Conclusion

Namespace-based segmentation in Portainer creates clear boundaries between teams, preventing accidental cross-team interference and enforcing the principle of least privilege. Map Portainer teams to Kubernetes namespaces, enable namespace isolation in Portainer settings, add Kubernetes NetworkPolicies for hard network isolation, and script namespace provisioning to ensure consistent configuration for every new team.
