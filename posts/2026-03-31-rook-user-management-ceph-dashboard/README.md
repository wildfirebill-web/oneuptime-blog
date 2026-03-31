# How to Configure User Management in the Ceph Dashboard

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Dashboard, User Management, Security

Description: Create and manage Ceph Dashboard users, roles, and permissions to enable team-based access control for Rook-managed cluster administration.

---

## Overview

The Ceph Dashboard supports multi-user access with role-based access control (RBAC). You can create users with specific roles (read-only, block-manager, etc.) to enable different team members to access only the sections they need.

## Default Admin User

The initial admin user is created by Rook during cluster bootstrap. Access credentials:

```bash
# Get admin username
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
  -o jsonpath='{.data.password}' | base64 --decode

# The default username is "admin"
```

## Built-in Dashboard Roles

The Dashboard ships with these built-in roles:

| Role | Permissions |
|---|---|
| administrator | Full access to all sections |
| read-only | View-only access, no changes |
| block-manager | Manage RBD images and pools |
| rgw-manager | Manage Object Gateway |
| cluster-manager | Manage OSDs, hosts, MONs |
| pool-manager | Create and delete pools |
| cephfs-manager | Manage CephFS filesystems |

## Creating a New Dashboard User

Navigate to Administration > User Management > Users, click "Create":

CLI equivalent:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-user-create \
  --enabled \
  alice \
  administrator

# Set password
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-user-set-password alice "SecurePassword123!"

# List users
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-user-show
```

## Creating Custom Roles

Define granular permissions with custom roles:

```bash
# Create a role that can only view pools and RBD
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-role-create dev-readonly

# Add read scopes to the role
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-role-add-scope-perms dev-readonly pool read

kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-role-add-scope-perms dev-readonly rbd-image read

# Assign role to a user
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-user-set-roles bob dev-readonly
```

Available scopes for permissions:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-scope-list
# Output includes: hosts, osd, mon, mgr, pool, pg, rbd-image, rbd-mirroring,
#                  iscsi, nfs-ganesha, cephfs, rgw, dashboard-settings, etc.
```

## Disable or Lock a User

```bash
# Disable user (prevent login)
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-user-disable alice

# Re-enable
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-user-enable alice

# Delete user
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-user-delete alice
```

## Force Password Change on Next Login

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph dashboard ac-user-set-info alice \
  --pwd-update-required true
```

## Summary

Ceph Dashboard RBAC allows assigning built-in roles (administrator, read-only, block-manager, etc.) or custom roles with granular scope-level permissions to each user. Using the `ac-user-create` and `ac-role-add-scope-perms` commands, you can implement least-privilege access for development, operations, and monitoring teams on the same dashboard.
