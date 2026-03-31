# How to Print User Keys with ceph auth print-key

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Authentication

Description: Learn how to extract raw secret keys from Ceph users using ceph auth print-key for use in application configuration and Kubernetes Secrets.

---

## What Is ceph auth print-key

`ceph auth print-key` prints only the raw base64-encoded secret key for a given Ceph authentication entity. Unlike `ceph auth get`, which returns the full keyring block including capabilities, `print-key` returns just the key value. This is useful when you need to inject the key directly into a configuration file, environment variable, or Kubernetes Secret.

Run from the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph auth print-key client.admin
```

## Basic Usage

Retrieve the key for any user:

```bash
ceph auth print-key client.myapp
```

Output:

```text
AQBzm7dg...==
```

The output is a single line with no trailing newline, making it ideal for scripting.

## Comparing print-key vs auth get

`ceph auth get` returns the full keyring block:

```text
[client.myapp]
    key = AQBzm7dg...==
    caps mon = "allow r"
    caps osd = "allow rw pool=appdata"
```

`ceph auth print-key` returns only:

```text
AQBzm7dg...==
```

Use `print-key` when you need just the secret key and `auth get` when you need the full keyring file.

## Creating Kubernetes Secrets from Keys

In Rook environments, application pods often need a Ceph secret key to authenticate. Create a Kubernetes Secret directly from the key:

```bash
# Export key as an environment variable
KEY=$(kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph auth print-key client.myapp)

# Create a Kubernetes Secret
kubectl create secret generic ceph-myapp-key \
  --from-literal=key="${KEY}" \
  -n myapp-namespace
```

## Using print-key in Application Configuration

Many Ceph clients accept the key directly in their configuration file. For example, a `ceph.conf` or a Kubernetes ConfigMap:

```ini
[client.myapp]
keyring = /etc/ceph/myapp.keyring
```

Or when the client accepts inline keys:

```ini
[client.myapp]
key = AQBzm7dg...==
```

Generate the key and pipe it into a config template:

```bash
KEY=$(ceph auth print-key client.myapp)
echo "[client.myapp]" > /etc/ceph/myapp.conf
echo "key = $KEY" >> /etc/ceph/myapp.conf
```

## Script: Export All Client Keys

If you need to audit or backup all client keys:

```bash
#!/bin/bash
for user in $(ceph auth ls --format json | jq -r '.auth_dump[] | select(.entity | startswith("client.")) | .entity'); do
  KEY=$(ceph auth print-key "$user")
  echo "$user: $KEY"
done
```

## Using with RBD and CephFS Mounts

When mounting RBD or CephFS volumes manually, the key is required:

```bash
# Mount CephFS using the key directly
mount -t ceph mon1:/ /mnt/cephfs \
  -o name=myapp,secret=$(ceph auth print-key client.myapp)
```

For Rook-managed PVCs, the CSI driver handles key injection automatically via Kubernetes Secrets. But for manual or non-Kubernetes workloads, `print-key` provides the raw value needed.

## Verifying Key Consistency

If you suspect a key mismatch between a Kubernetes Secret and the Ceph cluster, compare them:

```bash
# Key from Ceph
ceph auth print-key client.myapp

# Key from Kubernetes Secret
kubectl -n myapp-namespace get secret ceph-myapp-key \
  -o jsonpath='{.data.key}' | base64 -d
```

Both should match. If they differ, regenerate the Kubernetes Secret using the current Ceph key.

## Summary

`ceph auth print-key` prints only the raw secret key for a Ceph user, making it ideal for injecting credentials into application configurations, environment variables, and Kubernetes Secrets. Use it in scripts to automate credential distribution, and always compare the stored key with the Ceph value to detect drift in Rook-managed environments.
