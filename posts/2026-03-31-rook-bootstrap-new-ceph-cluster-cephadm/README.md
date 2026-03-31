# How to Bootstrap a New Ceph Cluster with cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, cephadm, Bootstrap, Cluster, Installation

Description: Bootstrap a new Ceph cluster from scratch using cephadm, the official Ceph cluster lifecycle manager, on a single node and expand from there.

---

## What Is cephadm?

`cephadm` is the official tool for deploying and managing Ceph clusters using containers. It orchestrates Ceph daemons as containerized services via systemd and Podman (or Docker), manages SSH-based host connections, and handles upgrades.

## Prerequisites

- RHEL/CentOS 8+, Ubuntu 20.04+, or Debian 10+
- Python 3 on all nodes
- Docker or Podman installed
- Passwordless SSH from the first node to all other nodes
- Minimum 8 GB RAM, 2 CPUs per node

## Step 1: Install cephadm

```bash
# Ubuntu/Debian
curl --silent --remote-name --location \
  https://github.com/ceph/ceph/raw/reef/src/cephadm/cephadm

chmod +x cephadm
sudo mv cephadm /usr/local/bin/

# Add the Ceph package repo
sudo cephadm add-repo --release reef
sudo cephadm install
```

## Step 2: Bootstrap the First Monitor

Run bootstrap on your first node. This starts the initial monitor and manager:

```bash
sudo cephadm bootstrap \
  --mon-ip 10.0.0.10 \
  --cluster-network 10.0.0.0/24 \
  --initial-dashboard-user admin \
  --initial-dashboard-password changeme
```

Output includes:
- Ceph Dashboard URL: `https://10.0.0.10:8443/`
- Admin credentials

## Step 3: Verify Bootstrap

```bash
ceph status
```

Expected output shows 1 mon, 1 mgr, and HEALTH_WARN (only 1 mon is expected at this stage).

## Step 4: Install ceph-common Tools

```bash
sudo cephadm install ceph-common
ceph -v
```

## Step 5: Add More Hosts

Before adding hosts, install the public SSH key:

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.0.0.11
ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.0.0.12

# Add hosts to the cluster
ceph orch host add node2 10.0.0.11
ceph orch host add node3 10.0.0.12
```

## Step 6: Deploy OSDs

Let cephadm use all available unpartitioned drives:

```bash
ceph orch apply osd --all-available-devices
```

Or target specific devices:

```bash
ceph orch daemon add osd node2:/dev/sdb
```

Monitor OSD deployment:

```bash
ceph orch ps | grep osd
ceph osd tree
```

## Step 7: Verify Cluster Health

```bash
ceph health detail
ceph df
```

Once you have at least 3 monitors and sufficient OSDs, the cluster should report `HEALTH_OK`.

## Summary

cephadm bootstraps a new Ceph cluster in minutes by starting the first monitor with `cephadm bootstrap --mon-ip`, then expanding by adding hosts via `ceph orch host add` and deploying OSDs with `ceph orch apply osd`. The built-in dashboard is immediately available after bootstrap for a graphical management interface.
