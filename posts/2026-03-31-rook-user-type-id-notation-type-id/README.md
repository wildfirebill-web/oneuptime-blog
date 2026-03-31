# How to Understand User Type and ID Notation (TYPE.ID) in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Learn how Ceph user identities work using TYPE.ID notation, how names map to keyring sections, and how to reference users correctly in commands.

---

## The TYPE.ID Format

Every Ceph authentication entity is identified using a two-part name in the format `TYPE.ID`. The type indicates what kind of entity it is, and the ID is a unique identifier within that type. This notation is used consistently across all Ceph authentication commands, configuration files, and keyrings.

Examples of valid TYPE.ID names:

```text
client.admin
client.myapp
osd.0
osd.14
mon.node1
mds.0
mgr.ceph-a
```

## Supported Types

Ceph defines the following entity types:

| Type | Description |
|---|---|
| `client` | External applications, admins, service accounts |
| `osd` | Object Storage Daemon |
| `mon` | Monitor daemon |
| `mds` | Metadata Server daemon |
| `mgr` | Manager daemon |

The `client` type is the most commonly used type for manually created users. Internal daemon types are used by Ceph's own service processes.

## How the Notation Appears in Keyrings

Keyring files store credentials using the TYPE.ID format as section headers:

```ini
[client.admin]
    key = AQA...==

[osd.0]
    key = AQB...==

[client.myapp]
    key = AQC...==
```

In Rook environments, these keyrings are stored as Kubernetes Secrets. For example:

```bash
kubectl -n rook-ceph get secret rook-ceph-admin-keyring -o jsonpath='{.data.keyring}' | base64 -d
```

Output:

```text
[client.admin]
    key = AQDef...==
    caps mds = "allow *"
    caps mgr = "allow *"
    caps mon = "allow *"
    caps osd = "allow *"
```

## Using TYPE.ID in ceph Commands

The full TYPE.ID must be used in most `ceph auth` commands:

```bash
# Get a specific user
ceph auth get client.myapp

# Delete a user
ceph auth del client.myapp

# Modify capabilities
ceph auth caps client.myapp mon 'allow r' osd 'allow rw pool=data'

# Export a user's keyring
ceph auth export client.myapp
```

For OSD and daemon users, use the same pattern:

```bash
ceph auth get osd.5
ceph auth caps osd.5 osd 'allow *' mon 'allow profile osd'
```

## Short Form for Client Users

Some Ceph commands and tools accept the short form for `client.*` users. When specifying a user in the `--name` or `-n` flag, you can omit the `client.` prefix in some contexts, but it is best practice to always include the full TYPE.ID:

```bash
# Full form - always correct
ceph --name client.myapp --keyring /etc/ceph/myapp.keyring health

# Short form - works in some commands
rados --id myapp --keyring /etc/ceph/myapp.keyring ls mypool
```

## ID Uniqueness Within a Type

IDs must be unique within each type, but the same ID can exist across different types. For example, `osd.0` and `client.0` are distinct entities:

```bash
ceph auth get osd.0   # OSD daemon with ID 0
ceph auth get client.0  # Client user with ID "0" (unusual but valid)
```

## Creating Users with the Correct Notation

Always specify the full TYPE.ID when creating users:

```bash
ceph auth get-or-create client.prometheus \
  mon 'allow r' \
  mgr 'allow r'
```

## Summary

Ceph authentication entities use a `TYPE.ID` format where type is one of `client`, `osd`, `mon`, `mds`, or `mgr`, and ID is a unique string within that type. Always use the full notation in commands like `ceph auth get`, `ceph auth del`, and `ceph auth caps`. Keyrings use this format as section headers, and Rook stores them as Kubernetes Secrets.
