# How to Configure Ceph Orchestrator with cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, cephadm, Orchestrator, Storage, Configuration

Description: Learn how to configure and use the Ceph orchestrator framework with cephadm to manage daemon placement, service specs, and cluster operations.

---

## What Is the Ceph Orchestrator

The Ceph orchestrator is a framework built into the Ceph manager that provides a standardized API for deploying and managing Ceph services. cephadm implements this orchestrator API, meaning all cluster operations (deploying monitors, OSDs, RGW, MDS) go through `ceph orch` commands regardless of the underlying container platform.

## Verifying Orchestrator Status

```bash
# Check which orchestrator backend is active
ceph orch status

# Expected output:
# Backend: cephadm
# Available: Yes
# Paused: No
```

If no orchestrator is configured:

```bash
# Enable cephadm as the orchestrator
ceph mgr module enable cephadm
ceph orch set backend cephadm
```

## Listing and Managing Services

```bash
# List all running services
ceph orch ls

# Show detailed service status
ceph orch ls --export

# List all daemon instances
ceph orch ps

# Filter by service type
ceph orch ps --daemon-type osd
ceph orch ps --daemon-type mon
```

## Applying Service Specifications

The orchestrator accepts declarative service specs:

```yaml
# mon-spec.yaml
service_type: mon
placement:
  hosts:
    - node1
    - node2
    - node3
```

```bash
# Apply from file
ceph orch apply -i mon-spec.yaml

# Apply inline
ceph orch apply mon "node1,node2,node3"
```

## Managing OSD Deployment

```bash
# List available devices
ceph orch device ls

# Deploy OSDs on all unused devices
ceph orch apply osd --all-available-devices

# Remove a specific OSD
ceph orch daemon rm osd.5 --force
```

Use a service spec for precise OSD placement:

```yaml
# osd-spec.yaml
service_type: osd
service_id: production-osds
placement:
  hosts:
    - node2
    - node3
    - node4
data_devices:
  paths:
    - /dev/sdb
    - /dev/sdc
db_devices:
  paths:
    - /dev/nvme0n1
```

```bash
ceph orch apply -i osd-spec.yaml
```

## Pausing and Resuming the Orchestrator

During maintenance, pause automatic reconciliation:

```bash
# Pause orchestrator (stops automatic daemon deployment)
ceph orch pause

# Perform maintenance
# ...

# Resume orchestrator
ceph orch resume
```

## Refreshing Host Inventory

cephadm periodically scans hosts for available devices and running daemons. Force a refresh:

```bash
# Refresh all hosts
ceph orch host refresh

# Refresh specific host
ceph orch host refresh node2
```

## Daemon Operations

```bash
# Restart a specific daemon
ceph orch daemon restart mon.node1

# Start/stop a daemon
ceph orch daemon stop osd.5
ceph orch daemon start osd.5

# Redeploy a daemon (pulls latest container image)
ceph orch daemon redeploy mon.node1
```

## Orchestrator Configuration

Tune orchestrator behavior:

```bash
# Set retry interval for failed daemon deployments
ceph config set mgr mgr/cephadm/retry_interval 60

# Configure container image pull timeout
ceph config set mgr mgr/cephadm/container_image_base \
  quay.io/ceph/ceph

# Enable/disable automatic OSD deployment
ceph config set mgr mgr/cephadm/autotune_memory_target_ratio 0.7
```

## Summary

The Ceph orchestrator with cephadm provides a unified, declarative API for managing all Ceph daemon deployments. Service specifications define the desired state, and the orchestrator continuously reconciles actual cluster state toward that specification. Pausing the orchestrator during maintenance, using service spec files for precise placement control, and monitoring service status through `ceph orch ls` and `ceph orch ps` are the core operational workflows for running a cephadm-managed production Ceph cluster.
