# How to Manage Namespace Access Control in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Access Control, RBAC, Multi-Tenancy

Description: Learn how to configure user and team access to specific Kubernetes namespaces in Portainer for multi-tenant cluster management.

## Overview

Portainer's access control for Kubernetes namespaces lets you restrict which Portainer users and teams can see and manage resources in each namespace. This enables multi-tenant clusters where different teams operate in isolated namespaces.

## Assigning Access to a Namespace

1. Select your Kubernetes environment.
2. Go to **Namespaces** and click on a namespace.
3. Click **Manage access**.
4. Add users or teams and assign a role:
   - **Standard user**: Can deploy and manage applications.
   - **Readonly user**: Can view resources but not modify them.
5. Click **Update**.

## Portainer Access Roles

| Role | Capabilities |
|------|-------------|
| **Environment Administrator** | Full access to everything in the environment |
| **Standard User** | Create, edit, delete own resources in authorized namespaces |
| **Readonly User** | View-only access to authorized namespaces |

## How Portainer Translates to Kubernetes RBAC

When you assign namespace access in Portainer, it creates Kubernetes RBAC resources:

```yaml
# Portainer creates a RoleBinding like this
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: portainer-access-john
  namespace: production
subjects:
  - kind: ServiceAccount
    name: john-portainer-sa
    namespace: portainer
roleRef:
  kind: ClusterRole
  name: portainer-namespace-user     # Portainer's custom ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

## Creating Custom Kubernetes RBAC Roles

For granular control beyond Portainer's built-in roles:

```yaml
# Custom role for a developer in the staging namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: staging-developer
  namespace: staging
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec"]
    verbs: ["get", "list", "watch", "create"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: staging-developer-binding
  namespace: staging
subjects:
  - kind: User
    name: jane@company.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: staging-developer
  apiGroup: rbac.authorization.k8s.io
```

## Verifying Access

```bash
# Check if a user can perform a specific action in a namespace
kubectl auth can-i create deployments \
  --namespace=staging \
  --as=jane@company.com

# List all role bindings in a namespace
kubectl get rolebindings --namespace=staging

# Check effective permissions for the current user
kubectl auth can-i --list --namespace=staging
```

## Conclusion

Portainer's namespace access control provides a user-friendly interface for multi-tenant Kubernetes environments. It translates team membership into Kubernetes RBAC rules, making it easy to grant the right level of access to the right teams without deep Kubernetes expertise.
