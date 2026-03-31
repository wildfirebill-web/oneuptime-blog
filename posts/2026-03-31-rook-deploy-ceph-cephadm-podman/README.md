# How to Deploy Ceph Using cephadm (Container-Based with Podman)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, cephadm, Podman, Deployment, Storage

Description: Learn how to bootstrap and manage a Ceph cluster using cephadm with Podman containers for a modern, declarative Ceph deployment outside of Kubernetes.

---

## What Is cephadm

cephadm is the official Ceph cluster lifecycle management tool introduced in Ceph Octopus. It runs all Ceph daemons as containers (using Podman or Docker), provides day-2 operations like upgrades and scaling, and integrates with the Ceph orchestration framework. Unlike Rook (which runs on Kubernetes), cephadm manages Ceph on bare-metal or VM hosts directly.

## Prerequisites

Each host needs:
- RHEL 8/9, CentOS Stream 8/9, Ubuntu 20.04+, or Debian 11+
- Podman 3.0+ or Docker
- Python 3.6+
- NTP configured
- SSH access between hosts

```bash
# Install cephadm on the bootstrap node
curl --silent --remote-name --location \
  https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm

chmod +x cephadm
sudo mv cephadm /usr/local/bin/

# Add Ceph repos
sudo cephadm add-repo --release quincy
sudo cephadm install
```

## Bootstrapping the First Monitor

Run bootstrap on the first host:

```bash
sudo cephadm bootstrap \
  --mon-ip 192.168.1.10 \
  --cluster-network 192.168.2.0/24 \
  --initial-dashboard-user admin \
  --initial-dashboard-password changeme123
```

This command:
- Pulls the Ceph container image
- Starts the first monitor and manager container
- Generates keyrings and `ceph.conf`
- Deploys the Ceph Dashboard
- Installs `ceph` CLI inside the container

After bootstrap:

```bash
# Verify cluster health
sudo cephadm shell -- ceph status
```

## Adding More Hosts

Distribute the SSH key and add hosts to the cluster:

```bash
# Copy the bootstrap SSH key to new hosts
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.11
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.12

# Add hosts to cephadm management
sudo cephadm shell -- ceph orch host add node2 192.168.1.11
sudo cephadm shell -- ceph orch host add node3 192.168.1.12

# Verify hosts
sudo cephadm shell -- ceph orch host ls
```

## Deploying OSDs

List available storage devices:

```bash
sudo cephadm shell -- ceph orch device ls
```

Deploy OSDs on all available drives:

```bash
sudo cephadm shell -- ceph orch apply osd --all-available-devices
```

Or target specific hosts and devices:

```bash
sudo cephadm shell -- ceph orch daemon add osd node2:/dev/sdb
sudo cephadm shell -- ceph orch daemon add osd node3:/dev/sdb
```

## Checking Container Status

cephadm manages daemon containers with Podman:

```bash
# List all Ceph containers on a node
sudo podman ps --filter label=ceph=true

# View logs for a specific daemon
sudo cephadm logs --name mon.node1

# Inspect a daemon
sudo cephadm inspect-image
```

## Verifying the Deployment

```bash
sudo cephadm shell -- ceph status
```

Expected healthy output:

```text
cluster:
  id:     a1b2c3d4-...
  health: HEALTH_OK

services:
  mon: 3 daemons, quorum node1,node2,node3
  mgr: node1(active)
  osd: 6 osds: 6 up, 6 in

data:
  pools:   1 pools, 1 pgs
  objects: 0 objects, 0 B
  usage:   75 MiB used, 18 TiB / 18 TiB avail
  pgs:     1 active+clean
```

## Summary

cephadm simplifies Ceph deployment by managing all daemons as Podman containers on standard Linux hosts. The `bootstrap` command handles initial monitor and manager setup, while `ceph orch` commands manage the full cluster lifecycle including adding hosts, deploying OSDs, and applying service specifications. For teams that need a production Ceph cluster without Kubernetes, cephadm with Podman offers a modern, container-native deployment model with strong operational tooling.
