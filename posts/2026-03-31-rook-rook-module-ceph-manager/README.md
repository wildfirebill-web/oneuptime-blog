# How to Set Up the Rook Module in Ceph Manager

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Kubernetes, Orchestrator, Module

Description: Learn how to enable and configure the Rook orchestrator module in Ceph Manager to manage Ceph daemon lifecycle through Kubernetes custom resources.

---

The Rook module is a Ceph Manager plugin that implements the Orchestrator interface using the Rook operator on Kubernetes. It enables the `ceph orch` command family to manage daemon deployment, placement, and scaling by translating commands into Kubernetes custom resource (CR) operations.

## Prerequisites

Before enabling the Rook module, ensure:

- A Rook-Ceph operator is deployed in your Kubernetes cluster
- The Ceph cluster was bootstrapped via Rook
- The active Ceph Manager pod has Kubernetes RBAC access to the `rook-ceph` namespace

## Enabling the Rook Module

Enable the module and set it as the active orchestrator backend:

```bash
ceph mgr module enable rook
ceph orch set backend rook
```

Verify the setup:

```bash
ceph orch status
```

Expected output:

```
Backend: rook
Available: Yes
```

## Configuring the Rook Module

Set the Kubernetes namespace where Rook is deployed:

```bash
ceph config set mgr mgr/rook/storage_class rook-ceph-block
```

The module auto-discovers the namespace from the Rook operator's environment when running inside Kubernetes.

## Listing Kubernetes-Managed Daemons

With the Rook module active, standard orchestrator commands show Kubernetes-managed daemons:

```bash
ceph orch ps
```

```
NAME        HOST    STATUS   REFRESHED
osd.0       node-1  running  30s ago
osd.1       node-2  running  30s ago
mon.a       node-1  running  1m ago
mgr.a       node-2  running  1m ago
```

## Deploying New OSDs

Trigger OSD deployment on all available devices:

```bash
ceph orch apply osd --all-available-devices
```

This creates or updates the `CephCluster` CR's storage section to include newly discovered devices.

## Scaling Monitors

Change the monitor count through the orchestrator:

```bash
ceph orch apply mon --placement="3"
```

The Rook operator reconciles the `CephCluster` CR and adjusts monitor pods accordingly.

## Checking Rook Operator Logs

When orchestrator operations fail, review the Rook operator logs:

```bash
kubectl logs -n rook-ceph deployment/rook-ceph-operator --tail=50
```

## Summary

The Rook Ceph Manager module connects the `ceph orch` interface to the Rook Kubernetes operator, enabling standard orchestrator commands to manage daemon deployment as Kubernetes custom resources. Enabling it requires setting the backend to `rook` and ensuring proper RBAC configuration, after which daemon scaling and placement operations translate directly into Kubernetes reconciliation events.
