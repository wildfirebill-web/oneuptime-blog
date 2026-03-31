# How to Use Object Prefix Matching for Ceph User Caps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Learn how to use object prefix matching in Ceph capability strings to restrict user access to objects with specific name prefixes within a pool.

---

## What Is Object Prefix Matching

Ceph OSD capabilities support object prefix matching, which allows you to restrict a user to accessing only objects whose names begin with a specified prefix. This is more granular than pool-level restrictions and allows multiple tenants to share a pool with object-name-based isolation.

## Syntax for Prefix Matching

Use the `object_prefix` keyword in the OSD capability string:

```bash
ceph auth get-or-create client.prefix-user \
  mon 'allow r' \
  osd 'allow rw pool=shared object_prefix tenant1-'
```

This user can only access objects whose names start with `tenant1-` in the `shared` pool.

## Practical Examples

Create separate users for different teams sharing a pool:

```bash
# Team A user - access to objects prefixed with "team-a-"
ceph auth get-or-create client.team-a \
  mon 'allow r' \
  osd 'allow rw pool=shared object_prefix team-a-'

# Team B user - access to objects prefixed with "team-b-"
ceph auth get-or-create client.team-b \
  mon 'allow r' \
  osd 'allow rw pool=shared object_prefix team-b-'
```

## Testing Prefix Restrictions

Create test objects and verify access:

```bash
# Create objects
echo "data" | rados -p shared put team-a-file1 -
echo "data" | rados -p shared put team-b-file1 -

# Verify team-a user can access its objects
rados --name client.team-a --keyring /tmp/team-a.keyring \
  -p shared get team-a-file1 /tmp/result
echo "team-a access to own objects: SUCCESS"

# Verify team-a user cannot access team-b objects
rados --name client.team-a --keyring /tmp/team-a.keyring \
  -p shared get team-b-file1 /tmp/result
```

Expected error for unauthorized access:

```text
RADOS returned error: -13 (Permission denied)
```

## Combining Prefix with Pool Restrictions

You can combine pool and object prefix restrictions:

```bash
ceph auth get-or-create client.scoped-user \
  mon 'allow r' \
  osd 'allow rw pool=prod object_prefix app1-, allow r pool=staging object_prefix app1-'
```

## Multiple Prefix Rules

Allow access to multiple prefixes:

```bash
ceph auth get-or-create client.multi-prefix \
  mon 'allow r' \
  osd 'allow rw pool=shared object_prefix logs-, allow r pool=shared object_prefix config-'
```

## Limitations of Prefix Matching

Prefix matching has some important limitations:

- It only matches on object name prefixes, not arbitrary substrings or suffixes
- Listing objects still shows all object names in the pool; only access to individual objects is restricted
- RBD and CephFS use their own naming conventions, making prefix matching complex for block or filesystem workloads
- Object prefix matching is primarily useful for raw RADOS workloads with controlled object naming

## Use Cases

Object prefix matching is best suited for:

- Log archival systems where each application writes objects with its own prefix
- Object storage tenants who agree on naming conventions
- Data tiering systems where objects from different sources have distinct prefixes

## Verifying Configured Caps

Inspect the user's caps after creation:

```bash
ceph auth get client.team-a
```

Expected output:

```text
[client.team-a]
    key = AQD...==
    caps mon = "allow r"
    caps osd = "allow rw pool=shared object_prefix team-a-"
```

## Summary

Ceph supports object prefix matching in OSD capability strings using the `object_prefix` keyword. This allows multiple users to share a pool while being restricted to objects whose names match their assigned prefix. Combine with pool restrictions for defense in depth. This pattern works best for raw RADOS workloads where object naming conventions are controlled by the application. For Rook RBD or CephFS workloads, pool-level or namespace-level restrictions are more practical.
