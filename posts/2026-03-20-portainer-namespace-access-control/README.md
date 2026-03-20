# How to Manage Namespace Access Control in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Namespaces, RBAC, Access Control, DevOps

Description: Learn how to configure namespace-level access control in Portainer to give teams appropriate permissions for their Kubernetes workloads.

## Introduction

Portainer Business Edition provides team-based access control for Kubernetes namespaces. You can grant teams read-only or read-write access to specific namespaces, preventing unauthorized changes while enabling self-service deployment. This guide covers configuring namespace access control in Portainer.

## Prerequisites

- Portainer BE with Kubernetes environment
- Namespaces created
- Teams configured in Portainer

## Access Control Model

Portainer uses a role-based model with namespace granularity:

```text
Admin → Full access to all environments and namespaces
Team (with access) → Can view/deploy to assigned namespaces
Team (without access) → Cannot see the namespace in Portainer
User (team member) → Inherits team's namespace access
```

## Step 1: Create Teams in Portainer

Before assigning namespace access, create teams:

1. Go to **Settings → Teams**
2. Click **+ Add team**
3. Create teams:

```text
Team name: backend-team     Description: Backend development team
Team name: frontend-team    Description: Frontend development team
Team name: ops-team         Description: Operations team
Team name: data-team        Description: Data engineering team
```

4. Add users to teams:
   - Click on a team
   - Click **+ Add user**
   - Select users to add

## Step 2: Assign Namespace Access

1. In Portainer, select the Kubernetes environment
2. Click **Namespaces** in the sidebar
3. Click on a namespace (e.g., **production**)
4. Go to the **Access control** section

Or navigate to the access control UI directly:
1. Click on a namespace
2. Find the **Namespace access** panel

## Step 3: Configure Team Access

Assign access levels to teams:

```text
Namespace: production
──────────────────────────────────────
Team: ops-team        → Full (Admin) access
Team: backend-team    → Read/Write access
Team: frontend-team   → Read/Write access (specific namespaces)
Team: data-team       → No access (they use different namespaces)
```

Access levels:
- **None** - Team cannot see the namespace
- **Read-only** - Can view resources but not create/update/delete
- **Read/Write** - Full CRUD on namespace resources
- **Admin** (some versions) - Can also manage quotas and access

## Step 4: Apply RBAC via Kubernetes Directly

Portainer translates its access control settings into Kubernetes RBAC:

```yaml
# Portainer creates these automatically when you configure team access

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: portainer-rw
  namespace: production
rules:
  - apiGroups: ["", "apps", "batch", "extensions", "autoscaling"]
    resources: ["*"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: portainer-rw-backend-team
  namespace: production
subjects:
  - kind: Group
    name: portainer-backend-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: portainer-rw
  apiGroup: rbac.authorization.k8s.io
```

## Step 5: Create Custom RBAC for Fine-Grained Control

For more specific access control, create custom RBAC resources:

```yaml
# Read-only role for specific resources
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-viewer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  # No create/update/delete permissions

---
# Developer role (can deploy but not delete)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
    # No delete permission
  - apiGroups: [""]
    resources: ["pods", "pods/log", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch"]
    # Can view but not modify services
```

## Step 6: Test Access Control

Verify access control works as expected:

```bash
# Test as a user in backend-team
# (Assuming service account setup for testing)
kubectl auth can-i create deployment \
  --namespace=production \
  --as=system:serviceaccount:portainer:backend-user

kubectl auth can-i create deployment \
  --namespace=staging \
  --as=system:serviceaccount:portainer:backend-user
# Should be: no (if staging is not accessible to backend-team)
```

## Step 7: Configure User-Level Access (Portainer BE)

In addition to team access, you can grant direct user access:

1. Navigate to a namespace access control panel
2. Switch to **Users** tab
3. Add individual users with specific access levels

This is useful for:
- Temporarily granting access during an incident
- Giving a consultant access to specific namespaces
- Service account-level access

## Step 8: Audit Access Configuration

```bash
# View all RBAC bindings in a namespace
kubectl get rolebindings -n production

# View a specific binding
kubectl describe rolebinding portainer-rw-backend-team -n production

# Check what a user can do
kubectl auth can-i --list \
  --namespace=production \
  --as=john.doe
```

## Conclusion

Namespace access control in Portainer BE provides a clean way to implement team-based Kubernetes multi-tenancy. Teams can only see and manage their assigned namespaces, preventing accidental changes to unrelated environments. For organizations with complex permission requirements, combine Portainer's team access with custom Kubernetes RBAC roles for fine-grained control over specific resource types.
