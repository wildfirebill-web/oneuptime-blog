# How to Handle Node Maintenance in Rook-Ceph (Cordon, Drain)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Node Maintenance, Storage

Description: Step-by-step guide to safely cordoning and draining Kubernetes nodes running Rook-Ceph storage without data loss or cluster health degradation.

---

## Why Node Maintenance Requires Extra Care in Rook-Ceph

Rook-Ceph deploys OSD, MON, MDS, and MGR daemons directly on Kubernetes nodes. Taking a node offline without preparation can cause Ceph OSDs to go down, triggering rebalancing, degraded placement groups, and potentially health errors. A structured maintenance procedure prevents these issues.

## Step 1 - Set the noout Flag Before Maintenance

Before taking a node offline, tell Ceph not to start rebalancing when OSDs go down:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd set noout
```

This prevents Ceph from starting PG recovery just because OSDs on the maintenance node temporarily disappear.

Verify the flag is set:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd dump | grep flags
```

Expected output:

```text
flags noout
```

## Step 2 - Cordon the Node

Prevent new workloads from being scheduled on the node:

```bash
kubectl cordon <node-name>
```

Verify:

```bash
kubectl get nodes
```

The node status should show `SchedulingDisabled`.

## Step 3 - Drain the Node

Evict all non-daemonset pods from the node. The `--ignore-daemonsets` flag is required because Rook uses DaemonSets for some components:

```bash
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=300
```

If pods have PodDisruptionBudgets (PDBs) blocking the drain, add a timeout:

```bash
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=300 \
  --timeout=600s
```

## Step 4 - Monitor Ceph Health During Drain

While draining, watch Ceph health to ensure it stays acceptable:

```bash
watch kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

You may see OSDs go down:

```text
osd: 9 osds: 8 up (since 2m), 9 in
```

This is expected if the node hosted OSDs. Because `noout` is set, no rebalancing occurs.

## Step 5 - Perform Maintenance

At this point the node is cordoned and drained. You can safely:
- Apply OS patches or kernel upgrades
- Replace hardware (non-OSD)
- Perform BIOS/firmware updates
- Reboot the node

```bash
# Example: reboot the node via SSH
ssh admin@<node-ip> sudo reboot
```

## Step 6 - Uncordon the Node

Once maintenance is complete and the node is back online:

```bash
kubectl uncordon <node-name>
```

Confirm the node is Ready:

```bash
kubectl get nodes
```

## Step 7 - Unset the noout Flag

Remove the `noout` flag so that Ceph resumes normal OSD health monitoring:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd unset noout
```

Wait for Ceph to report a healthy state:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

Expected:

```text
health: HEALTH_OK
```

## Handling MON and MGR Pods During Drain

Rook runs MON pods with scheduling constraints. If a MON is on the drained node, Rook's operator will reschedule it automatically after the node is cordoned and the pod is evicted. Monitor this:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon -w
```

Allow a few minutes for the new MON to join quorum before proceeding with other tasks.

## OSD Replacement vs. Temporary Maintenance

If you are replacing a failed disk (not just doing node maintenance), do NOT use `noout`. Instead, follow the OSD replacement procedure:

```bash
# Remove the OSD first
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd out <osd-id>
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd purge <osd-id> --yes-i-really-mean-it
```

## Summary

Safe node maintenance in Rook-Ceph requires setting `noout` before cordoning and draining, performing the maintenance, then uncordoning and unsetting `noout`. This sequence prevents unnecessary data rebalancing, keeps Ceph in an acceptable health state, and ensures your cluster recovers gracefully once the node returns to service.
