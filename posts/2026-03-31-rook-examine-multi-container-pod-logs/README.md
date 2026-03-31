# How to Examine Multi-Container Pod Logs in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Logging, Debugging, Pod

Description: Learn how to efficiently examine logs from multi-container Rook-Ceph pods including CSI provisioners and OSD pods with multiple sidecar containers.

---

## Overview

Rook-Ceph pods typically run multiple containers. A CSI provisioner pod may have 5-6 containers (provisioner, attacher, resizer, snapshotter, plugin, omap-generator). Knowing how to target specific containers is essential for effective troubleshooting.

## Listing Containers in a Pod

Before reading logs, identify which containers are in the pod:

```bash
kubectl get pod -n rook-ceph <pod-name> \
  -o jsonpath='{.spec.containers[*].name}'
```

Or with formatting:

```bash
kubectl get pod -n rook-ceph <pod-name> \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
```

Example output for a CSI RBD provisioner pod:

```text
csi-provisioner
csi-resizer
csi-attacher
csi-snapshotter
csi-rbdplugin
csi-omap-generator
liveness-prometheus
```

## Reading Logs from a Specific Container

Use the `-c` flag to target a specific container:

```bash
kubectl logs -n rook-ceph <pod-name> -c csi-rbdplugin
kubectl logs -n rook-ceph <pod-name> -c csi-provisioner
```

## Reading Logs from All Containers in a Pod

To see all containers' logs interleaved, use `--all-containers`:

```bash
kubectl logs -n rook-ceph <pod-name> --all-containers=true
```

Add prefix to identify container origin:

```bash
kubectl logs -n rook-ceph <pod-name> \
  --all-containers=true \
  --prefix=true
```

## Streaming Logs in Real Time

Follow logs from a specific container to watch live events:

```bash
kubectl logs -n rook-ceph \
  -l app=csi-rbdplugin-provisioner \
  -c csi-rbdplugin \
  -f
```

## Accessing Previous Container Logs (After Crash)

When a container has crashed and restarted, use `--previous`:

```bash
kubectl logs -n rook-ceph <pod-name> \
  -c csi-rbdplugin \
  --previous
```

This is critical for understanding what caused OOMKill or panic crashes.

## Logs for OSD Pods

OSD pods contain a main `osd` container and sometimes sidecar containers:

```bash
# List OSD containers
kubectl get pod -n rook-ceph -l app=rook-ceph-osd \
  -o jsonpath='{range .items[*]}{.metadata.name}: {range .spec.containers[*]}{.name} {end}{"\n"}{end}'

# Read the OSD daemon logs
kubectl logs -n rook-ceph \
  -l app=rook-ceph-osd \
  -c osd \
  --tail=200
```

## Filtering Logs for Specific Events

Combine `kubectl logs` with grep for targeted investigation:

```bash
# Find provisioning errors
kubectl logs -n rook-ceph \
  -l app=csi-rbdplugin-provisioner \
  -c csi-provisioner | grep -iE "error|fail" | tail -30

# Find volume ID references
kubectl logs -n rook-ceph \
  -l app=csi-rbdplugin-provisioner \
  --all-containers | grep "<volume-id>"
```

## Exporting Logs for Analysis

For offline analysis or sharing with support:

```bash
for container in csi-provisioner csi-rbdplugin csi-attacher; do
  echo "=== $container ===" >> provisioner-logs.txt
  kubectl logs -n rook-ceph \
    -l app=csi-rbdplugin-provisioner \
    -c $container >> provisioner-logs.txt 2>&1
done
```

## Summary

Multi-container Rook-Ceph pods require targeted log access using the `-c` flag to specify containers. List containers first with `kubectl get pod -o jsonpath`, then target the relevant sidecar based on the issue - use `csi-provisioner` for PVC creation failures, `csi-rbdplugin` for volume operation errors, and `--previous` to capture crash logs from restarted containers.
