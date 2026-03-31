# How to Filter Applications by Namespace in Portainer - Apps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Namespace, Application, DevOps

Description: Learn how to filter and view applications by namespace in Portainer to quickly find and manage workloads across a multi-namespace Kubernetes cluster.

## Introduction

In a Kubernetes cluster with multiple namespaces and dozens of applications, finding the right workload quickly is essential. Portainer provides namespace filtering throughout its interface, allowing you to focus on applications belonging to a specific team, environment, or purpose. This guide covers how to effectively use namespace filtering in Portainer.

## Prerequisites

- Portainer with a connected Kubernetes environment
- Multiple namespaces with deployed applications

## Step 1: Set the Active Namespace in Portainer

Portainer's Kubernetes interface has a namespace selector at the top of the screen:

1. Open your Kubernetes environment in Portainer
2. Look for the **Namespace** dropdown in the top navigation bar
3. Select a specific namespace (e.g., `production`)
4. All resource views (Applications, Services, ConfigMaps, etc.) now filter to that namespace

Available options:
```text
All namespaces      - Show resources from every namespace
default             - The default Kubernetes namespace
kube-system         - System components (hidden by default)
production          - Your production namespace
staging             - Your staging namespace
development         - Your development namespace
```

## Step 2: Filter Applications in the Applications List

Navigate to **Applications** in the sidebar:

1. Select your Kubernetes environment
2. Click **Applications** in the left sidebar
3. The applications list shows all deployed workloads
4. Use the namespace dropdown to filter by namespace
5. Use the search bar to filter by application name within the selected namespace

The list shows:
```text
Name          Namespace      Stack     Status    Created
my-api        production     -         Running   2 days ago
my-frontend   production     -         Running   2 days ago
redis         staging        -         Running   5 hours ago
postgres      development    -         Running   1 day ago
```

## Step 3: Use kubectl to Filter by Namespace

For command-line filtering alongside Portainer:

```bash
# List all deployments in a specific namespace

kubectl get deployments -n production

# List pods with namespace filter
kubectl get pods -n staging

# List all resources in a namespace
kubectl get all -n development

# List resources across all namespaces
kubectl get pods --all-namespaces
# or
kubectl get pods -A

# Filter by label within a namespace
kubectl get pods -n production -l app=my-api

# Get deployments with output showing namespace
kubectl get deployments -A -o wide
```

## Step 4: Set a Default Namespace for Your kubectl Context

To avoid specifying `-n namespace` on every command:

```bash
# Set default namespace for the current context
kubectl config set-context --current --namespace=production

# Verify the setting
kubectl config view --minify | grep namespace

# Now all commands use production by default
kubectl get pods    # Same as: kubectl get pods -n production

# Switch default namespace
kubectl config set-context --current --namespace=staging
```

## Step 5: Use Namespace Selectors in Portainer BE

Portainer Business Edition provides additional filtering capabilities:

1. **Environment-level namespace access** - Team members only see their assigned namespaces
2. **Namespace quick-switch** - Switch between namespaces without leaving the current view
3. **Cross-namespace search** - Search across all namespaces in the global search

For teams, this means:
- The backend team logs in and only sees `production` and `staging` (their assigned namespaces)
- The data team logs in and only sees `data-platform` namespace
- Admins see all namespaces

## Step 6: Filter Other Resources by Namespace

The namespace filter applies to all Kubernetes resource types in Portainer:

```text
Applications (Deployments, StatefulSets, DaemonSets)
Services
ConfigMaps
Secrets
Persistent Volume Claims
Ingresses
Pods (in the pod view)
```

Navigate to each section and the selected namespace persists across views.

## Step 7: Find Applications Across Namespaces via kubectl

```bash
# Find all deployments matching a name pattern across namespaces
kubectl get deployments -A | grep my-api

# Find pods by label across all namespaces
kubectl get pods -A -l app=my-api

# Find services exposing a specific port
kubectl get services -A -o json | \
  jq -r '.items[] | select(.spec.ports[].port == 8080) |
  "\(.metadata.namespace)/\(.metadata.name)"'

# Find recently deployed apps (sorted by creation time)
kubectl get deployments -A --sort-by=.metadata.creationTimestamp
```

## Step 8: Organize with Consistent Labels for Better Filtering

Labels enable filtering beyond just namespaces:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
  namespace: production
  labels:
    app: my-api
    team: backend
    environment: production
    tier: api
    version: v2.0.0
```

Filter by label in Portainer's search or kubectl:

```bash
# Filter by team label
kubectl get deployments -A -l team=backend

# Filter by multiple labels
kubectl get pods -n production -l team=backend,tier=api

# Filter by environment
kubectl get all -A -l environment=production
```

## Conclusion

Namespace filtering in Portainer keeps your workspace organized and focused. Use the namespace dropdown to limit views to specific environments or teams, leverage labels for fine-grained filtering, and set kubectl default namespaces to reduce repetitive flags. In multi-team clusters, Portainer BE's access control ensures each team only sees relevant namespaces automatically, making filtering a natural part of the workflow.
