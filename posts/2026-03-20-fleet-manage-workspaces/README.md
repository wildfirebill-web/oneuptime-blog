# How to Manage Fleet Workspaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Rancher, Kubernetes, Workspaces

Description: Learn how to create and manage Fleet workspaces to isolate GitOps deployments across different teams, environments, or organizational units.

## Introduction

Fleet workspaces provide namespace-level isolation for GitOps operations. Each workspace is a Kubernetes namespace that contains its own set of GitRepo resources, ClusterGroups, and Bundles. This isolation enables multi-tenant Fleet deployments where different teams can manage their own clusters and applications without interfering with each other.

This guide covers how to create workspaces, assign clusters to workspaces, and manage RBAC for workspace isolation.

## Prerequisites

- Fleet installed in Rancher
- Admin access to the Fleet manager cluster
- `kubectl` configured with cluster-admin privileges
- Basic understanding of Kubernetes RBAC

## Understanding Fleet Workspaces

In Fleet, workspaces are implemented as Kubernetes namespaces with a specific structure:

- **fleet-local**: The default workspace for the local Rancher cluster
- **fleet-default**: The default workspace for all downstream clusters
- **Custom workspaces**: User-defined namespaces for team or environment isolation

Each workspace has its own:
- GitRepo resources
- ClusterGroup resources
- Bundle resources
- RBAC policies

## Creating a New Workspace

### Step 1: Create the Namespace

```bash
# Create a new namespace to serve as a Fleet workspace
kubectl create namespace fleet-team-alpha

# Alternatively, use a manifest for reproducibility
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: fleet-team-alpha
  labels:
    # Label helps identify Fleet workspaces
    fleet.cattle.io/workspace: "true"
    team: alpha
EOF
```

### Step 2: Register Clusters to the Workspace

In Rancher, clusters are registered to a workspace through the UI or by moving them:

```bash
# Check which workspace a cluster belongs to
kubectl get cluster my-cluster -n fleet-default -o jsonpath='{.metadata.namespace}'

# Move a cluster to a different workspace by updating its namespace reference
# Note: This typically requires re-registration in Rancher
```

### Step 3: Create Resources in the Workspace

```yaml
# gitrepo-in-workspace.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: team-alpha-apps
  # Place the GitRepo in the team workspace
  namespace: fleet-team-alpha
spec:
  repo: https://github.com/team-alpha/k8s-configs
  branch: main
  targets:
    - clusterSelector: {}
```

```bash
kubectl apply -f gitrepo-in-workspace.yaml
```

## Configuring Workspace RBAC

### Creating a Workspace Admin Role

```yaml
# workspace-admin-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: fleet-workspace-admin
  namespace: fleet-team-alpha
rules:
  # Manage GitRepo resources
  - apiGroups: ["fleet.cattle.io"]
    resources: ["gitrepos"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Manage ClusterGroups
  - apiGroups: ["fleet.cattle.io"]
    resources: ["clustergroups"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Read-only access to Bundles (Fleet manages these)
  - apiGroups: ["fleet.cattle.io"]
    resources: ["bundles", "bundledeployments"]
    verbs: ["get", "list", "watch"]
```

```yaml
# workspace-admin-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-admin
  namespace: fleet-team-alpha
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: fleet-workspace-admin
subjects:
  # Bind to a specific user
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
  # Or bind to a group
  - kind: Group
    name: team-alpha
    apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f workspace-admin-role.yaml
kubectl apply -f workspace-admin-binding.yaml
```

### Creating a Read-Only Role

```yaml
# workspace-viewer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: fleet-workspace-viewer
  namespace: fleet-team-alpha
rules:
  - apiGroups: ["fleet.cattle.io"]
    resources: ["gitrepos", "clustergroups", "bundles", "bundledeployments"]
    verbs: ["get", "list", "watch"]
```

## Managing Multiple Workspaces

### Viewing All Workspaces

```bash
# List all Fleet workspaces (namespaces starting with fleet-)
kubectl get namespaces | grep fleet

# Or with custom label
kubectl get namespaces -l fleet.cattle.io/workspace=true
```

### Switching Between Workspaces

```bash
# List GitRepos in a specific workspace
kubectl get gitrepo -n fleet-team-alpha

# List all GitRepos across all workspaces
kubectl get gitrepo -A

# Get bundles in a specific workspace
kubectl get bundles -n fleet-team-alpha
```

## Using Rancher UI for Workspace Management

### Switching Workspaces in the UI

1. Navigate to **Continuous Delivery** in Rancher
2. In the top-right area, look for the **workspace selector** dropdown
3. Click to switch between `fleet-default`, `fleet-local`, or custom workspaces

### Creating a Workspace in Rancher

1. Navigate to **Continuous Delivery > Advanced > Workspaces**
2. Click **Create**
3. Enter a workspace name
4. Assign clusters to the workspace

## Workspace Isolation Best Practices

### Recommended Workspace Structure

```
fleet-local           # Local Rancher cluster management
fleet-default         # Shared/platform-wide deployments
fleet-team-frontend   # Frontend team's clusters and apps
fleet-team-backend    # Backend team's clusters and apps
fleet-team-data       # Data platform clusters
fleet-prod-only       # Production-only critical services
```

### Environment-Based Workspaces

```bash
# Create workspaces by environment
for env in development staging production; do
  kubectl create namespace "fleet-${env}" --dry-run=client -o yaml | \
    kubectl apply -f -

  # Label the namespace
  kubectl label namespace "fleet-${env}" \
    fleet.cattle.io/workspace="true" \
    environment="${env}"
done
```

## Conclusion

Fleet workspaces provide a clean isolation model for multi-team, multi-environment GitOps deployments. By using namespaces as workspaces and combining them with Kubernetes RBAC, you can give teams autonomy over their own deployments without risking interference with other teams. A thoughtful workspace design — whether team-based, environment-based, or a combination — forms the foundation of a scalable and secure Fleet deployment.
