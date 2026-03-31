# How to Diagnose PVCs Stuck in Pending State with Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Troubleshooting, Pvc, Csi

Description: Learn how to diagnose and resolve PersistentVolumeClaims stuck in Pending state in Rook-Ceph clusters by identifying CSI, provisioner, and cluster health issues.

---

## Overview

PVCs stuck in the Pending state are a common issue in Rook deployments. The root cause can range from misconfigured StorageClasses, unhealthy Ceph clusters, CSI provisioner failures, or insufficient cluster resources. This guide walks through a systematic diagnostic process.

## Step 1 - Check the PVC Events

Start by describing the PVC to see events:

```bash
kubectl describe pvc <pvc-name>
```

Look for events in the output:

```text
Events:
  Type     Reason                Age   From                                                                    Message
  ----     ------                ----  ----                                                                    -------
  Warning  ProvisioningFailed    30s   rook-ceph.rbd.csi.ceph.com_...  failed to provision volume: ...
```

Common messages and their meanings:
- `rpc error: code = Internal` - CSI driver internal error, check provisioner logs
- `timeout waiting for driver to create volume` - provisioner pod is unhealthy
- `pool does not exist` - StorageClass references a non-existent pool

## Step 2 - Check the CSI Provisioner Pods

Verify the CSI provisioner pods are running:

```bash
kubectl -n rook-ceph get pods -l app=csi-rbdplugin-provisioner
kubectl -n rook-ceph get pods -l app=csi-cephfsplugin-provisioner
```

Check provisioner logs for errors:

```bash
kubectl -n rook-ceph logs deploy/csi-rbdplugin-provisioner \
  -c csi-provisioner --tail=50
```

## Step 3 - Verify the StorageClass Configuration

Check the StorageClass referenced by the PVC:

```bash
kubectl get storageclass <storageclass-name> -o yaml
```

Verify:
- `provisioner` matches the deployed CSI driver name
- `clusterID` matches your Rook namespace
- `pool` name exists in Ceph
- Secret names and namespaces are correct

Check the referenced secrets exist:

```bash
kubectl -n rook-ceph get secret rook-csi-rbd-provisioner
kubectl -n rook-ceph get secret rook-csi-rbd-node
```

## Step 4 - Check Ceph Cluster Health

A degraded Ceph cluster may refuse to provision new volumes:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

If health is not `HEALTH_OK`, investigate the specific issue:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Common health issues that block provisioning:
- `HEALTH_WARN: too few PGs per OSD` - cluster needs more PGs
- `HEALTH_WARN: 1 pool(s) do not have an application enabled` - enable app on pool
- `HEALTH_ERR: insufficient standby MDS daemons` - MDS issues for CephFS

## Step 5 - Check Pool Exists and is Healthy

Verify the pool referenced in the StorageClass exists:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool ls
```

Check pool quota (if full, provisioning will fail):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph df detail
```

## Step 6 - Check CSI Node Plugin

For RBD volumes, the node plugin must be running on the scheduling node:

```bash
kubectl -n rook-ceph get daemonset rook-ceph-csi-rbdplugin
```

Check for pods on nodes where the workload is scheduled:

```bash
kubectl -n rook-ceph get pods -l app=csi-rbdplugin -o wide
```

## Step 7 - Enable CSI Debug Logging

If the above steps don't identify the issue, enable debug logging:

```bash
kubectl -n rook-ceph edit configmap rook-ceph-csi-config
```

Add debug log level:

```yaml
data:
  CSI_LOG_LEVEL: "5"
```

Restart the CSI driver pods and check logs again:

```bash
kubectl -n rook-ceph rollout restart deploy/csi-rbdplugin-provisioner
kubectl -n rook-ceph logs deploy/csi-rbdplugin-provisioner -c csi-rbdplugin
```

## Summary

Diagnosing PVCs stuck in Pending state in Rook involves checking PVC events, verifying CSI provisioner pod health, validating StorageClass configuration and secrets, confirming Ceph cluster health, and checking pool existence and quota. Work through these steps systematically, using `kubectl describe`, CSI provisioner logs, and the Ceph toolbox to identify the specific failure point.
