# How to Create a Ceph Monitor Recovery Runbook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Runbook, Recovery, Kubernetes

Description: A structured runbook for recovering failed Ceph monitors in Rook, covering quorum loss detection, monitor removal, and safe redeployment procedures.

---

## Why Monitor Recovery Is Critical

Ceph monitors maintain the cluster map. Losing quorum (fewer than majority of monitors reachable) makes the cluster read-only or completely unavailable. Quick recovery is essential.

## Step 1: Detect Monitor Problems

```bash
kubectl -n rook-ceph get pods | grep mon
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon stat
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph quorum_status | python3 -m json.tool
```

Look for monitors not in the quorum list. Check pod logs:

```bash
kubectl -n rook-ceph logs rook-ceph-mon-a-<pod-id> --tail=100
```

## Step 2: Assess Quorum State

A 3-monitor cluster needs 2 healthy monitors. A 5-monitor cluster needs 3.

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph quorum_status
```

If quorum is lost, the `ceph` commands will hang. Use `--connect-timeout` to avoid blocking:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph --connect-timeout 10 mon stat
```

## Step 3: Remove the Failed Monitor

If a monitor pod is crash-looping and cannot recover:

```bash
# Find the failing monitor name, e.g., mon-b
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon remove b
```

Then delete the Kubernetes resources:

```bash
kubectl -n rook-ceph delete deployment rook-ceph-mon-b
kubectl -n rook-ceph delete service rook-ceph-mon-b
kubectl -n rook-ceph delete pvc rook-ceph-mon-b  # if using PVCs
```

## Step 4: Force Quorum Recovery (Emergency)

If only one monitor remains and quorum is lost, use the inject monmap approach:

```bash
# Get the monmap from the surviving monitor
kubectl -n rook-ceph exec -it rook-ceph-mon-a-<pod> -- ceph-mon --extract-monmap /tmp/monmap

# Remove the failed monitor from the map
kubectl -n rook-ceph exec -it rook-ceph-mon-a-<pod> -- monmaptool /tmp/monmap --rm b

# Inject the modified monmap
kubectl -n rook-ceph exec -it rook-ceph-mon-a-<pod> -- ceph-mon --inject-monmap /tmp/monmap
```

## Step 5: Let Rook Redeploy the Monitor

After removing the failed monitor, trigger reconciliation:

```bash
kubectl -n rook-ceph annotate cephcluster rook-ceph \
  rook.io/do-not-reconcile- --overwrite
```

The Rook operator will schedule a new monitor pod on an available node.

## Step 6: Verify Recovery

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon stat
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
kubectl -n rook-ceph get pods | grep mon
```

All monitors should be `Running` and `in quorum`.

## Summary

Monitor recovery in Rook-Ceph requires identifying the failed monitor, removing it from quorum, cleaning up Kubernetes resources, and allowing the operator to redeploy. In extreme cases, monmap injection can restore a single-monitor cluster back to a working state.
