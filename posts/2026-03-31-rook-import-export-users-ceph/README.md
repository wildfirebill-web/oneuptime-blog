# How to Import and Export Users in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Learn how to export Ceph user keyrings for backup or migration and import them into another cluster using ceph auth export and ceph auth import.

---

## Why Import and Export Users

User import and export are useful for several scenarios:

- Migrating users from one Ceph cluster to another
- Backing up all authentication entities before a major upgrade
- Restoring users after accidental deletion
- Cloning a cluster's auth configuration to a new environment

## Exporting Users

`ceph auth export` writes one or all user keyrings to stdout or a file. Run from the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

Export all users to a file:

```bash
ceph auth export -o /tmp/all-keyrings.keyring
```

Export a single user:

```bash
ceph auth export client.myapp -o /tmp/myapp.keyring
```

The exported file is a standard keyring format:

```ini
[client.admin]
    key = AQA...==
    caps mds = "allow *"
    caps mgr = "allow *"
    caps mon = "allow *"
    caps osd = "allow *"

[client.myapp]
    key = AQB...==
    caps mon = "allow r"
    caps osd = "allow rw pool=appdata"
```

## Copying the Keyring from the Toolbox

Since the toolbox pod is ephemeral, copy the keyring file to your local machine or a Kubernetes ConfigMap:

```bash
# Copy from toolbox to local
kubectl -n rook-ceph cp rook-ceph-tools-<pod-id>:/tmp/all-keyrings.keyring ./all-keyrings.keyring

# Or save directly as a Kubernetes Secret
KEYRING=$(kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph auth export)
kubectl create secret generic ceph-auth-backup \
  --from-literal=keyring="$KEYRING" \
  -n rook-ceph
```

## Importing Users

`ceph auth import` reads a keyring file and adds or updates all entities found in it. Existing users with matching entity names are updated if the key or capabilities differ.

Import from a file:

```bash
ceph auth import -i /tmp/all-keyrings.keyring
```

Import a single user by importing a file containing only that user:

```bash
ceph auth import -i /tmp/myapp.keyring
```

## Verifying After Import

After importing, confirm the users were created or updated:

```bash
ceph auth ls | grep "client.myapp"
ceph auth get client.myapp
```

## Migration Workflow Between Clusters

To migrate users from Cluster A to Cluster B:

```bash
# On Cluster A - export all client users
ceph auth ls --format json | \
  jq -r '.auth_dump[] | select(.entity | startswith("client.")) | .entity' | \
  while read user; do
    ceph auth get "$user"
  done > /tmp/client-keyrings.keyring

# Copy the file to Cluster B (via ConfigMap, Secret, or scp)

# On Cluster B - import the keyrings
ceph auth import -i /tmp/client-keyrings.keyring
```

## Handling Conflicts During Import

If an entity already exists in the target cluster with a different key, the import will update it with the key from the file. To avoid overwriting existing users, use `ceph auth get-or-create` instead of import for individual users:

```bash
# Safer individual user creation (does not overwrite existing keys)
ceph auth get-or-create client.myapp \
  mon 'allow r' \
  osd 'allow rw pool=appdata'
```

## Backing Up Rook Auth State

Before Rook or Ceph upgrades, export all auth entities as a backup:

```bash
#!/bin/bash
DATE=$(date +%Y%m%d)
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph auth export > /backup/ceph-auth-${DATE}.keyring
echo "Auth backup saved to /backup/ceph-auth-${DATE}.keyring"
```

## Summary

`ceph auth export` backs up one or all user keyrings in keyring file format, and `ceph auth import` restores them. Use export before major upgrades or when migrating clusters. Verify imported users with `ceph auth get` after import. For non-destructive migrations, prefer `ceph auth get-or-create` over bulk import to avoid overwriting keys in the target cluster.
