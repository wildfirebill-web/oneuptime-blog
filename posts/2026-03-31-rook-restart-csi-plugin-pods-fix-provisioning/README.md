# How to Restart CSI Plugin Pods to Fix Provisioning in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CSI, Troubleshooting, Plugin

Description: Safely restart Rook-Ceph CSI plugin pods to resolve provisioning hangs and volume mount failures without disrupting running workloads.

---

## Overview

CSI plugin pods in Rook-Ceph can occasionally enter a degraded state where volume operations hang or fail. Restarting these pods is often the fastest way to restore normal provisioning without affecting already-mounted volumes on running pods.

## Understanding CSI Pod Types

Before restarting, understand what each pod type handles:

- **csi-rbdplugin DaemonSet**: Handles volume mounting/unmounting on each node. Restarting affects only new mount operations on that node.
- **csi-cephfsplugin DaemonSet**: Same as above for CephFS volumes.
- **csi-rbdplugin-provisioner Deployment**: Handles PVC creation/deletion. Restarting delays provisioning until the new pod is ready.
- **csi-cephfsplugin-provisioner Deployment**: Same for CephFS provisioning.

## Checking Current Pod Status

```bash
kubectl get pods -n rook-ceph | grep csi
```

Identify problematic pods - look for `CrashLoopBackOff`, `Error`, or pods with many restarts:

```bash
kubectl get pods -n rook-ceph | grep csi | \
  awk '{print $1, $3, $4}' | column -t
```

## Restarting Provisioner Pods

For PVC provisioning issues, restart the provisioner deployment:

```bash
kubectl rollout restart deployment \
  csi-rbdplugin-provisioner \
  csi-cephfsplugin-provisioner \
  -n rook-ceph
```

Monitor the rollout:

```bash
kubectl rollout status deployment/csi-rbdplugin-provisioner -n rook-ceph
```

## Restarting Node Plugin Pods

For volume mount failures on a specific node, restart the DaemonSet pod on that node:

```bash
# Find the pod on the affected node
kubectl get pods -n rook-ceph -o wide | grep csi-rbdplugin | grep <node-name>

# Delete the specific pod (DaemonSet recreates it automatically)
kubectl delete pod -n rook-ceph csi-rbdplugin-<pod-id>
```

To restart all CSI node plugin pods across the cluster:

```bash
kubectl rollout restart daemonset \
  csi-rbdplugin \
  csi-cephfsplugin \
  -n rook-ceph
```

## Impact on Running Workloads

Restarting CSI node plugin pods is generally safe for pods with already-mounted volumes because:

1. The actual mount is handled by the kernel (kmod rbd / ceph)
2. The CSI plugin re-registers with kubelet on restart
3. Existing bind mounts are not affected

However, pods attempting to mount a new volume at the time of restart may see a brief delay or need to retry.

## Verifying Recovery

After restart, test that provisioning is restored:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
EOF

kubectl get pvc csi-test-pvc -w
```

If the PVC binds, provisioning is restored. Clean up:

```bash
kubectl delete pvc csi-test-pvc
```

## Automating Pod Restarts for Recurring Issues

If CSI pods require frequent manual restarts, investigate the underlying cause:

```bash
kubectl logs -n rook-ceph -l app=csi-rbdplugin-provisioner \
  -c csi-rbdplugin --previous | tail -50
```

Frequent restarts indicate resource exhaustion, Ceph connectivity problems, or bugs that need addressing at the source.

## Summary

Restarting CSI plugin pods in Rook-Ceph resolves many transient provisioning and mount issues without impacting running workloads. Use `kubectl rollout restart deployment` for provisioner pods and `kubectl delete pod` for targeted node-level plugin restarts. Always investigate logs for root causes if restarts become a recurring fix.
