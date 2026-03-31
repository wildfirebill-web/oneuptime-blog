# How to Fix 'mon is not in quorum' in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Quorum, Troubleshooting, High Availability

Description: Restore Ceph monitor quorum by diagnosing split-brain conditions, recovering crashed monitors, and ensuring monitor count meets quorum requirements.

---

## Introduction

Ceph monitors use the Paxos protocol and require a majority quorum to function. With 3 monitors, 2 must be healthy. When quorum is lost, the entire cluster becomes unavailable - no reads or writes can proceed. This guide covers how to diagnose and restore monitor quorum.

## Identifying the Problem

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example:

```text
HEALTH_ERR mon.b is not in quorum; mon.c is not in quorum
2 mons are not in quorum
```

Check monitor status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon stat
```

List all monitors and their quorum membership:

```bash
ceph quorum_status | python3 -m json.tool
```

## Step 1 - Check Monitor Pod Status

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon
kubectl -n rook-ceph describe pod rook-ceph-mon-b-<pod-id>
kubectl -n rook-ceph logs rook-ceph-mon-b-<pod-id> --previous
```

## Step 2 - Check Node Availability

If the node hosting a monitor is down:

```bash
kubectl get nodes
```

If the node is `NotReady`, Kubernetes will eventually reschedule the monitor pod. You may need to manually evict it:

```bash
kubectl drain <node-name> --delete-emptydir-data --force --ignore-daemonsets
```

## Step 3 - Force a Single Monitor (Emergency)

If only one monitor is available, temporarily force it to operate alone:

```bash
# Access the remaining monitor pod
kubectl -n rook-ceph exec -it rook-ceph-mon-a-<pod-id> -- bash

# Inside the pod, force quorum with just this monitor
ceph-mon --extract-monmap /tmp/monmap
monmaptool --print /tmp/monmap
monmaptool --rm b /tmp/monmap
monmaptool --rm c /tmp/monmap
ceph-mon --inject-monmap /tmp/monmap
```

Then restart the monitor:

```bash
kubectl -n rook-ceph delete pod rook-ceph-mon-a-<pod-id>
```

## Step 4 - Recover Crashed Monitor Data

If a monitor's data store is corrupted:

```bash
# Check the monitor store
kubectl -n rook-ceph exec -it rook-ceph-mon-b-<pod-id> -- \
  ceph-mon --mkfs --monmap /tmp/monmap --keyring /etc/ceph/keyring
```

## Step 5 - Remove a Permanently Failed Monitor

After new monitor is added to replace a dead one:

```bash
ceph mon remove b
ceph mon add c <new-ip>:6789
```

## Preventing Quorum Loss

Always run an odd number of monitors (3 or 5) for resilience:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
spec:
  mon:
    count: 3
    allowMultiplePerNode: false
```

Ensure monitors are distributed across different nodes:

```yaml
placement:
  mon:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: rook-ceph-mon
        topologyKey: kubernetes.io/hostname
```

## Summary

Monitor quorum loss halts the entire Ceph cluster. Recovery involves identifying which monitors are down, restarting pods if nodes are available, or using the emergency single-monitor recovery procedure for worst-case scenarios. Running 3 or 5 monitors distributed across separate nodes with pod anti-affinity prevents most quorum loss scenarios.
