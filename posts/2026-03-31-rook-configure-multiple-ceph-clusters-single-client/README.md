# How to Configure Multiple Ceph Clusters on a Single Client

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Client, Multi-Cluster, Configuration, Storage

Description: Learn how to configure a single client node to connect to multiple independent Ceph clusters using different config files and keyrings.

---

It is common to need access to multiple independent Ceph clusters from a single client - for example, connecting to both a production and a staging cluster, or managing a disaster recovery setup. Ceph supports this through separate config files and keyrings.

## How Ceph Identifies Clusters

Each Ceph cluster has a unique FSID. By default, tools look for `/etc/ceph/ceph.conf` and the matching keyring. To target a different cluster, you specify an alternate config file or use the `--cluster` flag.

## Using --cluster Flag

If you name your cluster something other than `ceph` in the config file, Ceph tools use the filename convention `<cluster>.conf` and `<cluster>.client.<user>.keyring`:

```bash
# Default cluster (reads /etc/ceph/ceph.conf)
ceph -s

# Secondary cluster (reads /etc/ceph/prod.conf)
ceph --cluster prod -s
```

## Setting Up Config Files

Create separate config files for each cluster:

```ini
# /etc/ceph/ceph.conf  (cluster 1 - dev)
[global]
fsid = aaaaaaaa-1111-2222-3333-444444444444
mon_host = 192.168.1.10, 192.168.1.11, 192.168.1.12
auth_cluster_required = cephx
```

```ini
# /etc/ceph/prod.conf  (cluster 2 - production)
[global]
fsid = bbbbbbbb-5555-6666-7777-888888888888
mon_host = 10.0.1.10, 10.0.1.11, 10.0.1.12
auth_cluster_required = cephx
```

## Keyrings per Cluster

Each cluster needs its own keyring file:

```bash
# Default cluster keyring
ls /etc/ceph/ceph.client.admin.keyring

# Production cluster keyring (named prod.client.admin.keyring)
ls /etc/ceph/prod.client.admin.keyring
```

Fetch keyrings from each cluster's admin node:

```bash
# From dev cluster
scp dev-admin:/etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring

# From prod cluster (use cluster name prefix)
scp prod-admin:/etc/ceph/ceph.client.admin.keyring /etc/ceph/prod.client.admin.keyring
```

## Using RBD with Multiple Clusters

```bash
# List images on dev cluster
rbd -p mypool ls

# List images on prod cluster
rbd --cluster prod -p mypool ls

# Map an RBD image from each cluster
rbd map mypool/myimage
rbd --cluster prod map mypool/prodimage
```

## Using CephFS with Multiple Clusters

```bash
# Mount dev CephFS
mount -t ceph 192.168.1.10:/ /mnt/dev-cephfs \
  -o name=admin,secretfile=/etc/ceph/dev-secret

# Mount prod CephFS
mount -t ceph 10.0.1.10:/ /mnt/prod-cephfs \
  -o name=admin,secretfile=/etc/ceph/prod-secret
```

## Using Environment Variables

Avoid repetitive flags with environment variables:

```bash
export CEPH_CONF=/etc/ceph/prod.conf
export CEPH_KEYRING=/etc/ceph/prod.client.admin.keyring
ceph -s
rbd -p mypool ls
```

Reset to default cluster:

```bash
unset CEPH_CONF CEPH_KEYRING
```

## Summary

Configuring a single client for multiple Ceph clusters requires separate `.conf` files and keyrings for each cluster. Use the `--cluster <name>` flag to target a specific cluster, or set the `CEPH_CONF` environment variable for session-level context switching. This approach scales to as many clusters as needed without any interference between them.
