# How to Create Users with ceph auth add in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Learn how to create Ceph authentication users using ceph auth add, set capabilities, and manage the resulting keyrings in Rook environments.

---

## Overview of ceph auth add

`ceph auth add` creates a new Ceph authentication entity with specified capabilities. If the entity already exists, the command returns the existing entry without modifying it. This makes it safe to run repeatedly in automation scripts.

Access the Ceph CLI from the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

## Basic Syntax

```bash
ceph auth add <entity> [<capabilities>...]
```

Create a client user with read-only monitor access and read-write access to a specific pool:

```bash
ceph auth add client.myapp \
  mon 'allow r' \
  osd 'allow rw pool=mypool'
```

## Output and Key Generation

Ceph automatically generates a random key for the new user. After running `ceph auth add`, retrieve the user details:

```bash
ceph auth get client.myapp
```

Sample output:

```text
[client.myapp]
    key = AQD...==
    caps mon = "allow r"
    caps osd = "allow rw pool=mypool"
```

## Creating Users with Multiple Pool Access

Grant access to multiple pools by using multiple `allow` clauses separated by commas:

```bash
ceph auth add client.multipool \
  mon 'allow r' \
  osd 'allow rw pool=pool1, allow rw pool=pool2'
```

## Creating Read-Only Users

For monitoring or backup agents that only need to read data:

```bash
ceph auth add client.readonly \
  mon 'allow r' \
  osd 'allow r'
```

## Creating Admin-Level Users

For full administrative access (use sparingly):

```bash
ceph auth add client.myservice \
  mon 'allow *' \
  osd 'allow *' \
  mds 'allow *' \
  mgr 'allow *'
```

## Saving the Keyring to a File

Export the newly created user's keyring to a file for distribution:

```bash
ceph auth get client.myapp -o /tmp/myapp.keyring
```

In Rook environments, copy this to a Kubernetes Secret:

```bash
kubectl create secret generic ceph-myapp-keyring \
  --from-file=keyring=/tmp/myapp.keyring \
  -n myapp-namespace
```

## Difference Between auth add and auth get-or-create

| Command | Behavior if entity exists |
|---|---|
| `auth add` | Returns existing entry, no modification |
| `auth get-or-create` | Returns existing entry, no modification |
| `auth get-or-create-key` | Returns only the key |

Both `auth add` and `auth get-or-create` behave similarly when an entity already exists. The key difference is that `auth get-or-create` is more commonly used in scripts because it explicitly communicates the idempotent intent. Use `auth add` when you are performing initial user setup interactively.

## Verifying the Created User

After creation, always verify the user exists with the correct capabilities:

```bash
ceph auth ls | grep "client.myapp"
ceph auth get client.myapp
```

Also verify the key is accessible:

```bash
ceph auth print-key client.myapp
```

## Automation Script Example

Create multiple application users in a loop:

```bash
#!/bin/bash
USERS=("client.app1" "client.app2" "client.app3")
POOLS=("pool1" "pool2" "pool3")

for i in "${!USERS[@]}"; do
  ceph auth add "${USERS[$i]}" \
    mon 'allow r' \
    osd "allow rw pool=${POOLS[$i]}"
  echo "Created ${USERS[$i]} for ${POOLS[$i]}"
done
```

## Summary

`ceph auth add` creates a Ceph authentication entity with specified capabilities and a randomly generated key. It is idempotent - if the entity already exists, it returns the existing entry. Use `ceph auth get` after creation to verify the user and its capabilities, and export the keyring with `ceph auth get -o` for distribution to applications or Kubernetes Secrets in Rook environments.
