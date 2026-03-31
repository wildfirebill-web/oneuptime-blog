# How to Troubleshoot Container Storage Issues with Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, CSI, Container Storage, Kubernetes

Description: Diagnose and fix common container storage failures with Ceph in Kubernetes, including PVC stuck in Pending, volume mount errors, and CSI driver issues.

---

Container storage problems with Ceph often manifest as PVCs stuck in Pending state, pods unable to start due to volume attachment failures, or I/O errors inside containers. This guide provides a systematic approach to diagnosing these issues.

## Common Symptoms and First Steps

Start with broad status checks:

```bash
# Check overall Ceph cluster health
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status

# Check CSI driver pods
kubectl -n rook-ceph get pods | grep csi

# Check PVC status
kubectl get pvc -A | grep -v Bound
```

## PVC Stuck in Pending

If a PVC is in `Pending` state, describe it for error details:

```bash
kubectl describe pvc myapp-data -n mynamespace
```

Common causes and fixes:

**StorageClass not found:**
```bash
kubectl get storageclass
# Ensure the name in PVC matches an existing StorageClass
```

**No available OSDs or pool doesn't exist:**
```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool ls
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd stat
```

**CSI provisioner not running:**
```bash
kubectl -n rook-ceph logs deploy/csi-rbdplugin-provisioner -c csi-provisioner --tail=50
```

## Volume Mount Failures

If a pod is stuck in `ContainerCreating`, check events:

```bash
kubectl describe pod myapp-pod -n mynamespace | tail -30
```

Check the CSI node plugin on the node where the pod is scheduled:

```bash
NODE=$(kubectl get pod myapp-pod -o jsonpath='{.spec.nodeName}')
kubectl -n rook-ceph get pods -l app=csi-rbdplugin --field-selector spec.nodeName=$NODE

kubectl -n rook-ceph logs <csi-rbdplugin-pod> -c csi-rbdplugin --tail=100
```

Check if the RBD kernel module is loaded:

```bash
kubectl debug node/$NODE -it --image=ubuntu -- lsmod | grep rbd
```

## I/O Errors Inside Containers

If an application reports I/O errors, check if the Ceph cluster is degraded:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg stat
```

Check for slow requests:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd perf | sort -k3 -rn | head -10
```

## Volume Expansion Failures

If expanding a PVC fails:

```bash
kubectl describe pvc myapp-data | grep -E "Conditions|Events"

# Check if expansion is allowed in StorageClass
kubectl get storageclass rook-ceph-block -o yaml | grep allowVolumeExpansion
```

Manually trigger expansion if stuck:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd resize --size 20480 replicapool/csi-vol-<uuid>
```

## Collect Diagnostic Information

Gather all relevant information for filing a bug:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph report > /tmp/ceph-report.txt

kubectl -n rook-ceph get events --sort-by='.lastTimestamp' > /tmp/k8s-events.txt

kubectl -n rook-ceph logs deploy/rook-ceph-operator --tail=200 > /tmp/operator-logs.txt
```

## Summary

Container storage issues with Ceph usually fall into three categories: provisioning failures (CSI provisioner problems), attachment failures (kernel module or CSI node plugin issues), and I/O errors (cluster degradation). Start with `ceph status` and `kubectl describe pvc` to identify the category, then follow the specific diagnostic steps for CSI driver logs and cluster health to find the root cause.
