# How to Fix "noout flag is set" Warning in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Storage, Troubleshooting, OSD

Description: Learn how to diagnose and resolve the "noout flag is set" warning in Ceph that prevents OSDs from being marked out during failures.

---

## Understanding the noout Flag

The `noout` flag in Ceph is an operational flag that prevents OSDs from being automatically marked "out" when they go down. Normally, if an OSD remains down for more than `mon_osd_down_out_interval` (default 600 seconds), Ceph will mark it out and begin rebalancing data. Setting `noout` disables this automatic behavior.

While `noout` is a useful maintenance tool, leaving it set accidentally causes Ceph to show a warning:

```text
HEALTH_WARN
    noout flag(s) set
```

This warning indicates Ceph is operating in a non-standard mode where failed OSDs will never be marked out, which can prevent data recovery if a real OSD failure occurs.

## Why noout Gets Set

The `noout` flag is commonly set during:

- Node maintenance or hardware replacement
- Kubernetes node drain operations
- Cluster upgrades to prevent unnecessary rebalancing
- Manual troubleshooting sessions

The problem arises when operators forget to unset it after maintenance completes.

## Checking the Current Flag State

Use `ceph osd dump` or `ceph status` to check which flags are set:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd dump | grep flags
```

Expected output showing the flag is set:

```text
flags noout,sortbitwise,recovery_deletes,purged_snapdirs,pglog_hardlimit
```

You can also check with:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

```text
HEALTH_WARN noout flag(s) set
    OSDMAP_FLAGS noout flag(s) set
```

## Unsetting the noout Flag

To clear the `noout` flag and restore normal OSD out behavior:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd unset noout
```

After unsetting, verify the flag is cleared:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd dump | grep flags
```

```text
flags sortbitwise,recovery_deletes,purged_snapdirs,pglog_hardlimit
```

## Verifying Cluster Health Recovery

After unsetting `noout`, check that the warning clears:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

```text
cluster:
  id:     abc123
  health: HEALTH_OK

services:
  mon: 3 daemons, quorum a,b,c (age 2h)
  mgr: a(active, since 3h)
  osd: 9 osds: 9 up, 9 in
```

## Safely Using noout During Maintenance

If you intentionally need `noout` for maintenance, use a controlled workflow:

```bash
# Before maintenance - set noout
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd set noout

# Perform your maintenance work here
# (drain node, replace hardware, etc.)

# After maintenance - always unset noout
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd unset noout

# Verify health is restored
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health
```

## Automating noout Management with Scripts

Create a helper script to manage the flag during Kubernetes node drains:

```bash
#!/bin/bash
# ceph-node-drain.sh - Safely drain a node with noout protection

NODE=$1
NAMESPACE="rook-ceph"

if [ -z "$NODE" ]; then
  echo "Usage: $0 <node-name>"
  exit 1
fi

echo "Setting Ceph noout flag..."
kubectl -n $NAMESPACE exec -it deploy/rook-ceph-tools -- ceph osd set noout

echo "Draining node $NODE..."
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data

echo "Maintenance window - press Enter when complete..."
read

echo "Uncordoning node $NODE..."
kubectl uncordon $NODE

echo "Unsetting Ceph noout flag..."
kubectl -n $NAMESPACE exec -it deploy/rook-ceph-tools -- ceph osd unset noout

echo "Verifying cluster health..."
kubectl -n $NAMESPACE exec -it deploy/rook-ceph-tools -- ceph health detail
```

## Checking OSD States After Unsetting noout

After removing the flag, verify OSDs have recovered their expected states:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
```

```text
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         2.72800  root default
-3         0.90933      host node1
 0    hdd  0.45456          osd.0       up   1.00000  1.00000
 1    hdd  0.45477          osd.1       up   1.00000  1.00000
```

All OSDs should show `up` and `in` status once maintenance is complete.

## Summary

The "noout flag is set" warning in Ceph is resolved by running `ceph osd unset noout` after maintenance is complete. Always pair `ceph osd set noout` with a corresponding `ceph osd unset noout` to avoid leaving the cluster in a degraded warning state. Use scripted workflows to ensure the flag is always cleared after node maintenance or upgrade operations.
