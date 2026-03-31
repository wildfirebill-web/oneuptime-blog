# How to Migrate from ceph-deploy to cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Migration, cephadm, Deployment Tool

Description: Learn how to migrate a Ceph cluster deployed with the legacy ceph-deploy tool to cephadm for modern containerized management and lifecycle automation.

---

ceph-deploy was the original Ceph deployment tool but is no longer maintained. cephadm is the current standard for deploying and managing Ceph, providing containerized daemons, automated upgrades, and a built-in orchestration layer.

## Understanding the Difference

| Feature | ceph-deploy | cephadm |
|---------|-------------|---------|
| Status | Deprecated | Active |
| Daemon style | Native packages | Containers |
| Upgrade automation | Manual | Built-in |
| Day-2 operations | Manual | Orchestrated |
| Rook integration | No | Via import |

## Pre-Migration Checklist

```bash
# Check current cluster health
ceph -s
ceph health detail

# Document current layout
ceph osd tree
ceph mon dump
ceph mgr dump

# Check Ceph version (requires Nautilus+)
ceph version
```

## Step 1 - Install cephadm on the Bootstrap Node

```bash
curl --silent --remote-name \
  --location https://github.com/ceph/ceph/raw/reef/src/cephadm/cephadm

chmod +x cephadm
sudo mv cephadm /usr/local/bin/
cephadm version
```

## Step 2 - Adopt the Existing Cluster

cephadm provides an adopt command for ceph-deploy clusters:

```bash
cephadm adopt --style legacy --name mon.a
```

Run this on each host that has a Ceph daemon managed by ceph-deploy.

## Step 3 - Import the Cluster Configuration

```bash
# Copy existing config and keyring
cp /etc/ceph/ceph.conf /etc/ceph/ceph.conf.backup
cephadm bootstrap \
  --mon-ip <mon-ip> \
  --allow-overwrite \
  --skip-monitoring-stack
```

## Step 4 - Adopt Remaining Daemons

For each OSD host:

```bash
# SSH to each OSD host and adopt
cephadm adopt --style legacy --name osd.0
cephadm adopt --style legacy --name osd.1
```

Verify daemons are running under cephadm:

```bash
cephadm ls
```

## Step 5 - Enable the Orchestrator

```bash
ceph mgr module enable orchestrator
ceph orch set backend cephadm
ceph orch status
```

## Step 6 - Verify Cluster Health

```bash
ceph -s
ceph orch ps
ceph orch host ls
```

## Step 7 - Enable Modern cephadm Features

```bash
# Enable telemetry (optional)
ceph mgr module enable telemetry

# Enable autoscaling
ceph mgr module enable pg_autoscaler

# View service spec
ceph orch ls
```

## Post-Migration Cleanup

Remove old ceph-deploy files:

```bash
rm -rf ~/ceph-deploy-*.log
pip uninstall ceph-deploy
```

## Summary

Migrating from ceph-deploy to cephadm uses the built-in adopt command to convert legacy native-package daemons to containerized cephadm-managed daemons one at a time. This approach keeps the cluster running throughout the migration while transitioning to the modern management interface. Enabling the orchestrator backend completes the migration and unlocks automated day-2 operations.
