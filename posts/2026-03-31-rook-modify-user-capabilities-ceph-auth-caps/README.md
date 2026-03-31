# How to Modify User Capabilities with ceph auth caps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Learn how to modify Ceph user capabilities using ceph auth caps to update permissions without deleting and recreating the user or its key.

---

## Overview of ceph auth caps

`ceph auth caps` replaces all capabilities for an existing Ceph authentication entity. It does not modify the user's key - only the capability strings are updated. This is the correct command to use when you need to change what a user can access without disrupting applications currently using the existing key.

Access from the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

## Basic Syntax

```bash
ceph auth caps <entity> <subsystem> '<capabilities>' [<subsystem> '<capabilities>'...]
```

## Updating OSD Capabilities

Add read-write access to an additional pool:

```bash
# Before: only appdata pool
ceph auth get client.myapp
# caps osd = "allow rw pool=appdata"

# After: both appdata and logs pools
ceph auth caps client.myapp \
  mon 'allow r' \
  osd 'allow rw pool=appdata, allow rw pool=logs'
```

## Restricting Existing Permissions

Downgrade a user from full access to pool-restricted access:

```bash
# Restrict a previously unrestricted user
ceph auth caps client.oldapp \
  mon 'allow r' \
  osd 'allow rw pool=apppool'
```

## Granting MDS Access

Add CephFS metadata server capabilities to an existing user:

```bash
ceph auth caps client.myapp \
  mon 'allow r' \
  mds 'allow rw' \
  osd 'allow rw tag cephfs data=myfs'
```

## Removing Capabilities from a Subsystem

To remove all capabilities for a specific subsystem, set it to an empty string:

```bash
# Remove MDS access but keep mon and osd caps
ceph auth caps client.myapp \
  mon 'allow r' \
  mds '' \
  osd 'allow rw pool=appdata'
```

Note: Setting a subsystem to an empty string effectively removes it from the entity's capability set.

## Verifying the Change

Always verify the caps after modification:

```bash
ceph auth get client.myapp
```

Expected output after the update:

```text
[client.myapp]
    key = AQD...==
    caps mon = "allow r"
    caps osd = "allow rw pool=appdata, allow rw pool=logs"
```

## Important Behavior: Full Replacement

`ceph auth caps` performs a full replacement of all capabilities for the entity. If you specify only `osd`, the `mon` caps are removed. Always specify ALL subsystems you want the user to have:

```bash
# Wrong: this removes mon caps entirely
ceph auth caps client.myapp \
  osd 'allow rw pool=appdata'

# Correct: specify all subsystems
ceph auth caps client.myapp \
  mon 'allow r' \
  osd 'allow rw pool=appdata'
```

## Updating Rook-Created CSI Users

Rook creates CSI users like `client.csi-rbd-node`. In rare cases, you may need to update their caps after a pool rename or new pool creation:

```bash
ceph auth caps client.csi-rbd-node \
  mon 'profile rbd' \
  osd 'profile rbd pool=rbd, profile rbd pool=newpool' \
  mgr 'allow rw'
```

Always consult the Rook documentation to verify the required capability format before modifying CSI user caps.

## Script: Bulk Cap Update

Update multiple users to add access to a new pool:

```bash
#!/bin/bash
NEW_POOL="analytics"
USERS=("client.app1" "client.app2")

for user in "${USERS[@]}"; do
  # Get current osd caps
  CURRENT=$(ceph auth get "$user" --format json | jq -r '.[0].caps.osd')
  # Append new pool
  NEW_CAPS="${CURRENT}, allow rw pool=${NEW_POOL}"
  ceph auth caps "$user" \
    mon 'allow r' \
    osd "$NEW_CAPS"
  echo "Updated $user"
done
```

## Summary

`ceph auth caps` replaces the full capability set for a Ceph user without modifying the key. Always specify all subsystems in the command because any subsystem omitted will have its caps removed. This is the safest way to update user permissions in production as it preserves the existing key and does not interrupt connected applications.
