# How to Deploy Red Hat Ceph Storage with cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Red Hat, Cephadm, Deployment, Storage

Description: Step-by-step guide to deploying Red Hat Ceph Storage using cephadm, covering bootstrap, adding hosts, OSDs, and verifying cluster health.

---

`cephadm` is the official deployment tool for Red Hat Ceph Storage (RHCS) and upstream Ceph. It uses containers (Podman or Docker) and SSH to manage cluster daemons without requiring Ansible or manual package installs on each node.

## Prerequisites

- RHEL 8 or 9 nodes with valid RHCS subscriptions
- Passwordless SSH access from the bootstrap node to all other nodes
- Podman installed on all nodes
- Firewall ports open: 3300, 6789 (monitors), 6800-7300 (OSDs)

## Step 1 - Install cephadm and Bootstrap

On the first node:

```bash
dnf install -y cephadm
cephadm bootstrap \
  --mon-ip 192.168.1.10 \
  --cluster-network 192.168.2.0/24 \
  --registry-url registry.redhat.io \
  --registry-username <RH_USERNAME> \
  --registry-password <RH_PASSWORD>
```

This creates the first monitor, manager, and a local Ceph config. At the end, cephadm prints the dashboard URL and credentials.

## Step 2 - Add the Ceph CLI

```bash
cephadm shell -- ceph status
```

Or install the ceph CLI directly:

```bash
cephadm add-repo --release reef
dnf install -y ceph-common
```

## Step 3 - Add Hosts

Copy the cephadm SSH key to each additional node and register them:

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.11
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.12

ceph orch host add node2 192.168.1.11
ceph orch host add node3 192.168.1.12
```

Verify hosts are visible:

```bash
ceph orch host ls
```

## Step 4 - Deploy Monitors

Add labels to control monitor placement, then apply:

```bash
ceph orch host label add node1 mon
ceph orch host label add node2 mon
ceph orch host label add node3 mon
ceph orch apply mon --placement "label:mon"
```

## Step 5 - Deploy OSDs

Let cephadm discover and use all available devices:

```bash
ceph orch apply osd --all-available-devices
```

Or target specific devices with a service spec:

```yaml
service_type: osd
service_id: default_drive_group
placement:
  host_pattern: "*"
data_devices:
  paths:
    - /dev/sdb
    - /dev/sdc
```

```bash
ceph orch apply -i osd_spec.yaml
```

## Step 6 - Verify Cluster Health

```bash
ceph status
ceph osd tree
ceph df
```

A healthy cluster shows `HEALTH_OK` and all OSDs `up` and `in`.

## Summary

Deploying Red Hat Ceph Storage with cephadm involves bootstrapping the first node, registering additional hosts over SSH, and applying service specs for monitors and OSDs. The containerized approach ensures consistent daemon versions and simplifies day-2 operations like upgrades and node additions.
