# How to Configure Reclaim Space Jobs with Rook CSI-Addons

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CSI, Storage, Kubernetes

Description: Learn how to create and configure ReclaimSpaceJob resources with Rook CSI-Addons to recover unused space in RBD and CephFS volumes.

---

## What Is a ReclaimSpaceJob

When files are deleted from an RBD block volume, the freed space is not automatically returned to the Ceph pool. The filesystem inside the RBD image marks blocks as free, but Ceph still allocates space for them. A `ReclaimSpaceJob` triggers an operation - either `fstrim` for block volumes or a sparsify operation - that communicates freed blocks back to Ceph, reclaiming that space in the pool. This is important for clusters where storage efficiency matters.

## Prerequisites

The CSI-Addons controller must be deployed and the CSI-Addons sidecar must be running in the Rook CSI provisioner. Verify:

```bash
kubectl get crd reclaimspacejobs.csiaddons.openshift.io
kubectl -n rook-ceph get pods -l app=csi-rbdplugin-provisioner -o jsonpath='{.items[0].spec.containers[*].name}'
```

The provisioner pod should list `csi-addons` among its containers.

## Creating a ReclaimSpaceJob

Create a `ReclaimSpaceJob` that targets a specific PVC:

```yaml
apiVersion: csiaddons.openshift.io/v1alpha1
kind: ReclaimSpaceJob
metadata:
  name: reclaim-my-volume
  namespace: my-app
spec:
  target:
    persistentVolumeClaim: my-rbd-pvc
  backOffLimit: 6
  retryDeadlineSeconds: 900
```

- `target.persistentVolumeClaim` - the PVC to reclaim space from
- `backOffLimit` - number of retry attempts on failure
- `retryDeadlineSeconds` - maximum time in seconds for the job to complete

Apply it:

```bash
kubectl apply -f reclaim-job.yaml
```

## Monitoring Job Progress

Check the job status:

```bash
kubectl get reclaimspacejob reclaim-my-volume -n my-app -o wide
```

Example output:

```text
NAME                  RESULT     CONDITIONS   AGE
reclaim-my-volume     Succeeded               45s
```

For failed jobs, inspect the conditions:

```bash
kubectl describe reclaimspacejob reclaim-my-volume -n my-app
```

The `Conditions` section will show why the operation failed, such as the volume not being mounted or the CSI driver not supporting the operation.

## Reclaim Space for Node-Attached Volumes

For RBD volumes currently mounted on a node, the reclaim operation runs `fstrim` against the mounted filesystem. The volume must be in use (mounted) for this path. For volumes not currently mounted, Rook uses `sparsify` on the raw image:

```bash
# Check if the PVC is currently mounted
kubectl -n my-app get pod -o jsonpath='{.items[*].spec.volumes[*].persistentVolumeClaim.claimName}'
```

## Viewing Reclaim Space Results in Ceph

After a successful ReclaimSpaceJob, verify the pool's used bytes decreased:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph df
```

Compare the `USED` column for the pool before and after. The change depends on how much data was deleted from the volume before the job ran.

## Bulk Reclaim with Multiple Jobs

To reclaim space on all PVCs in a namespace, create a job per PVC:

```bash
for pvc in $(kubectl get pvc -n my-app -o jsonpath='{.items[*].metadata.name}'); do
  kubectl apply -n my-app -f - <<EOF
apiVersion: csiaddons.openshift.io/v1alpha1
kind: ReclaimSpaceJob
metadata:
  name: reclaim-${pvc}
  namespace: my-app
spec:
  target:
    persistentVolumeClaim: ${pvc}
EOF
done
```

## Summary

`ReclaimSpaceJob` resources from CSI-Addons let you return freed filesystem space back to the Ceph pool after deletions. Create the job with a target PVC, configure retry limits, and monitor its status via `kubectl get reclaimspacejob`. For ongoing space hygiene, use `ReclaimSpaceCronJob` (a scheduled version) to run reclaim operations automatically on a schedule.
