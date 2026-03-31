# How to Troubleshoot Rook-Ceph Monitor Quorum Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Quorum, Troubleshooting

Description: Diagnose and fix Rook-Ceph monitor quorum issues using kubectl and ceph CLI tools to restore cluster availability.

---

Ceph monitors (mons) maintain the authoritative map of the cluster state. A quorum requires a majority of monitors to agree - with three monitors, at least two must be healthy. When quorum is lost, the entire Ceph cluster becomes unavailable, blocking all reads and writes.

## Identifying Quorum Loss

The first sign is usually that applications cannot read or write data. Check cluster health immediately:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

A cluster with quorum problems will show output like:

```text
health: HEALTH_ERR
        mon a is low on available space
        3 monitors have not enabled msgr2
        quorum_age: 1234
```

Or in severe cases:

```text
2024/01/15 10:23:45.123 7f... cluster [ERR] HEALTH_ERR: no active mgr
```

List all monitors and their status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon stat
```

```text
e12: 3 mons: a,b,c, election epoch 24, leader a, quorum a,b,c
```

If a monitor is missing from the quorum list, it has failed.

## Checking Monitor Pod Status

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon
```

```text
NAME                  READY   STATUS    RESTARTS   AGE
rook-ceph-mon-a-xxx   2/2     Running   0          5d
rook-ceph-mon-b-xxx   0/2     Pending   0          2m
rook-ceph-mon-c-xxx   2/2     Running   0          5d
```

A pending or crash-looping mon pod means that monitor is not contributing to quorum. Investigate logs:

```bash
kubectl -n rook-ceph logs rook-ceph-mon-b-xxx -c mon --previous
```

## Common Causes and Fixes

**Disk full on monitor node:** Monitors store data in `dataDirHostPath`. If the host disk is full, the monitor cannot write and will crash.

```bash
# On the affected node
df -h /var/lib/rook
du -sh /var/lib/rook/mon-b
```

Free up disk space or expand the volume, then delete the mon pod to let it restart.

**Network partition:** Check if nodes can reach each other. Monitors communicate on port 6789 (v1) and 3300 (v2).

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon dump
```

```text
epoch 12
fsid xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
last_changed 2024-01-15T10:00:00.000000+0000
created 2024-01-01T00:00:00.000000+0000
min_mon_release 17 (quincy)
election_strategy: 1
0: [v2:192.168.1.10:3300/0,v1:192.168.1.10:6789/0] mon.a
1: [v2:192.168.1.11:3300/0,v1:192.168.1.11:6789/0] mon.b
2: [v2:192.168.1.12:3300/0,v1:192.168.1.12:6789/0] mon.c
```

**Clock skew:** Ceph monitors are sensitive to time drift. Ensure NTP is synchronized:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph time-sync-status
```

If clock skew exceeds 0.05 seconds, the monitor will refuse to join quorum.

## Forcing a Monitor Restart

If a monitor pod is stuck in an error state, force restart it:

```bash
kubectl -n rook-ceph delete pod rook-ceph-mon-b-xxx
```

The Rook operator will recreate the pod automatically. Watch the recovery:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon -w
```

## Monitoring Quorum Continuously

Set up a watch to track quorum health during troubleshooting:

```bash
watch -n 5 "kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph quorum_status --format json-pretty"
```

The output shows which monitors are in the quorum and who the current leader is.

## Summary

Monitor quorum issues in Rook-Ceph are diagnosed by checking pod status, reviewing logs, and using `ceph mon stat` and `ceph quorum_status` commands. Common root causes include full disks, network partitions, and clock skew. Most issues resolve by addressing the underlying cause and restarting the affected monitor pod. If quorum cannot be restored automatically, the `restore-quorum` command is available as an emergency measure.
