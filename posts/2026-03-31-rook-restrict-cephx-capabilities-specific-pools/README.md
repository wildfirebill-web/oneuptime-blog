# How to Restrict CephX Capabilities to Specific Pools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephX, Pool, Access Control, Security

Description: Apply the principle of least privilege to Ceph by restricting CephX client capabilities to specific pools, namespaces, and object prefixes in Rook.

---

Granting `allow *` to all CephX clients is a common security anti-pattern. By scoping capabilities to specific pools, namespaces, and object prefixes, you enforce separation between applications sharing the same Ceph cluster.

## Pool-Level Restrictions

Create a key scoped to a single pool:

```bash
# Read-write access to one pool only
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.app-a \
  mon 'allow r' \
  osd 'allow rw pool=app-a-data'

# Read access to production, read-write to staging
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.app-b \
  mon 'allow r' \
  osd 'allow r pool=app-a-data, allow rw pool=app-b-data'
```

## Namespace Restrictions

Ceph supports namespace-level isolation within a pool:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.tenant-x \
  mon 'allow r' \
  osd 'allow rw pool=shared-pool namespace=tenant-x'
```

## Object Prefix Restrictions

Restrict access to objects with a specific prefix - useful for RBD images:

```bash
# Allow access only to objects with prefix "rbd_"
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.rbd-limited \
  mon 'profile rbd' \
  osd 'allow class-read object_prefix rbd_children, allow rwx pool=rbd-pool'
```

## Command-Level Restrictions on Monitors

Restrict which monitor commands a client can run:

```bash
# Read-only monitoring client
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.monitoring \
  mon 'allow r, allow command "status", allow command "health", allow command "pg stat", allow command "osd stat"' \
  mgr 'allow r'
```

## Verify Restrictions Are Working

Test that a restricted key cannot access unauthorized pools:

```bash
# Export the restricted key
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-key client.app-a > /tmp/app-a.key

# Try to access an unauthorized pool - should fail
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph --id app-a --key $(cat /tmp/app-a.key) \
  osd pool stats app-b-data 2>&1
```

Expected output: `Error EACCES: access denied`

## Review All Key Capabilities

Audit all keys for over-permissive capabilities:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth ls | grep -A3 "allow \*"
```

Replace any `allow *` OSD capabilities with explicit pool names.

## Summary

Restricting CephX capabilities to specific pools, namespaces, and object prefixes enforces application isolation in multi-tenant Ceph clusters. Use the minimum required capability for each client: `allow r` for read-only, `allow rw pool=specific-pool` for write access, and `profile rbd` for RBD clients. Regular capability audits catch over-permissive keys before they become a security incident.
