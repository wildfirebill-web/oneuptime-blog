# How to Get User Info with ceph auth get and ceph auth export

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Learn how to retrieve Ceph user details and export keyrings using ceph auth get and ceph auth export commands in Rook-managed clusters.

---

## Overview

Ceph provides two commands for retrieving information about a specific authentication entity: `ceph auth get` and `ceph auth export`. Both show user details, but they differ in output format and typical use cases.

Access these commands from the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

## Using ceph auth get

`ceph auth get` retrieves the full details of a specific user including their key and capability strings:

```bash
ceph auth get client.myapp
```

Sample output:

```text
[client.myapp]
    key = AQC...==
    caps mon = "allow r"
    caps osd = "allow rw pool=appdata"
```

The output is in keyring format, which is the same format used by keyring files on disk.

## Using ceph auth get for Daemon Users

System daemon users follow the same syntax:

```bash
ceph auth get osd.0
ceph auth get mon.node1
ceph auth get mgr.ceph-mgr-a
```

## Saving Output to a Keyring File

You can redirect the output of `ceph auth get` directly to a keyring file:

```bash
ceph auth get client.myapp -o /etc/ceph/myapp.keyring
```

This creates or overwrites `/etc/ceph/myapp.keyring` with the keyring content. This is the standard way to distribute credentials to application hosts.

## Using ceph auth export

`ceph auth export` is similar to `ceph auth get` but is designed for exporting the keyring to a file. With no arguments, it exports all entities:

```bash
# Export all users to stdout
ceph auth export

# Export all users to a file
ceph auth export -o /tmp/all-keyrings.keyring

# Export a specific user
ceph auth export client.myapp
```

The output format is identical to `ceph auth get`:

```text
[client.myapp]
    key = AQC...==
    caps mon = "allow r"
    caps osd = "allow rw pool=appdata"
```

## Difference Between get and export

| Feature | auth get | auth export |
|---|---|---|
| Single user | Yes | Yes |
| All users | No | Yes (no args) |
| Output format | Keyring | Keyring |
| Typical use | Inspect a user | Backup/migrate users |

## Extracting the Key Only

If you only need the secret key value (for example to configure an application), use `ceph auth print-key` instead:

```bash
ceph auth print-key client.myapp
```

Output:

```text
AQC...==
```

## Exporting to Kubernetes Secret

In Rook environments, you might want to create a Kubernetes Secret from a user's keyring for use by an application pod:

```bash
# Get the keyring content
KEYRING=$(ceph auth get client.myapp)

# Create a Kubernetes Secret
kubectl create secret generic ceph-myapp-keyring \
  --from-literal=keyring="$KEYRING" \
  -n myapp-namespace
```

Or using the key directly:

```bash
KEY=$(ceph auth print-key client.myapp)
kubectl create secret generic ceph-myapp-secret \
  --from-literal=key="$KEY" \
  -n myapp-namespace
```

## JSON Output

For scripting, export in JSON format:

```bash
ceph auth get client.myapp --format json
```

Sample output:

```json
[
  {
    "entity": "client.myapp",
    "key": "AQC...==",
    "caps": {
      "mon": "allow r",
      "osd": "allow rw pool=appdata"
    }
  }
]
```

## Summary

`ceph auth get` retrieves a single user's keyring details in keyring file format, and accepts `-o` to write directly to a file. `ceph auth export` is similar but can export all users when called with no user argument. Use `ceph auth print-key` when you only need the raw secret key value for application configuration or Kubernetes Secrets.
