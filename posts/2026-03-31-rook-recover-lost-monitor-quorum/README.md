# How to Recover from Lost Monitor Quorum in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Recovery, Disaster Recovery

Description: Step-by-step guide to recovering a Ceph cluster from lost monitor quorum using Rook, including emergency single-monitor recovery procedures.

---

## What Is Monitor Quorum Loss?

Ceph monitors use the Paxos consensus algorithm. A cluster with 3 monitors needs 2 to form quorum. If more than half of monitors fail simultaneously (e.g., 2 of 3 fail), the cluster loses quorum. In this state, clients cannot read or write data, and the cluster is effectively frozen. Recovering quorum is a critical emergency procedure.

## Diagnosing Quorum Loss

MON pods will be in `CrashLoopBackOff` or failing. Check:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon
kubectl -n rook-ceph logs rook-ceph-mon-a-<suffix> | tail -50
```

Attempt to connect to a surviving monitor:

```bash
kubectl -n rook-ceph exec -it rook-ceph-mon-a-<suffix> -- \
  ceph -s --connect-timeout 10
```

If the command hangs or returns connection errors, quorum is lost.

## Recovery with Remaining Monitor

If one monitor is still running, you can rebuild quorum by injecting a new monmap that contains only the surviving monitor:

```bash
# Stop all MON pods
kubectl -n rook-ceph scale deploy rook-ceph-mon-b --replicas=0
kubectl -n rook-ceph scale deploy rook-ceph-mon-c --replicas=0

# Extract the current monmap from the surviving monitor's data directory
kubectl -n rook-ceph exec -it rook-ceph-mon-a-<suffix> -- \
  ceph-mon --extract-monmap /tmp/monmap --mon-data /var/lib/ceph/mon/ceph-a

# Print the monmap
kubectl -n rook-ceph exec -it rook-ceph-mon-a-<suffix> -- \
  monmaptool --print /tmp/monmap
```

Remove failed monitors from the monmap:

```bash
kubectl -n rook-ceph exec -it rook-ceph-mon-a-<suffix> -- \
  monmaptool /tmp/monmap --rm b

kubectl -n rook-ceph exec -it rook-ceph-mon-a-<suffix> -- \
  monmaptool /tmp/monmap --rm c
```

Inject the modified monmap:

```bash
kubectl -n rook-ceph exec -it rook-ceph-mon-a-<suffix> -- \
  ceph-mon --inject-monmap /tmp/monmap --mon-data /var/lib/ceph/mon/ceph-a
```

Restart the surviving monitor:

```bash
kubectl -n rook-ceph scale deploy rook-ceph-mon-a --replicas=1
```

## Letting Rook Rebuild Monitors

After establishing single-monitor quorum, trigger Rook to add new monitors by confirming the `CephCluster` spec:

```bash
kubectl -n rook-ceph edit cephcluster rook-ceph
```

Ensure `mon.count: 3` is set. Rook will detect that only 1 monitor is running and create 2 more automatically.

## Verifying Recovery

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph status

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph quorum_status
```

The `quorum` array should list all 3 monitors.

## Summary

Recovering from lost monitor quorum requires injecting a modified monmap that contains only surviving monitors, then restarting the cluster from a single-monitor state. Rook handles rebuilding the remaining monitors once quorum is re-established. This procedure should be tested in staging environments before a production emergency occurs.
