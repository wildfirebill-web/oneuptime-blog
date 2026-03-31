# How to Fix 'clock skew detected' in Ceph Monitors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, Monitor, Clock Skew, NTP, Time Sync

Description: Fix clock skew errors in Ceph monitors by configuring NTP synchronization on Kubernetes nodes and adjusting Ceph's clock drift tolerance.

---

## Introduction

Ceph monitors use Paxos consensus, which requires all monitor nodes to have synchronized clocks. If clock skew exceeds 0.05 seconds by default, Ceph reports a health warning or error. This guide explains how to diagnose and fix clock synchronization issues.

## Identifying Clock Skew

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Output:

```
HEALTH_WARN clock skew detected on mon.b, mon.c
mon.b clock skew 0.102 > max 0.050
```

## Checking Clock State on Nodes

Identify which nodes are running monitor pods:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon -o wide
```

SSH to each monitor node and check time synchronization:

```bash
# Check chrony status
chronyc tracking
chronyc sources -v

# Or check NTP with timedatectl
timedatectl status
```

Look for large offset values or "NO" under "NTP synchronized".

## Fixing Clock Sync with chrony

On nodes using chronyd:

```bash
# Restart chrony to force resync
systemctl restart chronyd

# Force immediate sync
chronyc makestep

# Verify offset is low
chronyc tracking | grep "System time"
```

## Fixing Clock Sync with systemd-timesyncd

```bash
# Check status
timedatectl status

# Restart timesyncd
systemctl restart systemd-timesyncd

# Set NTP server
timedatectl set-ntp true
```

Configure a reliable NTP server in `/etc/systemd/timesyncd.conf`:

```ini
[Time]
NTP=pool.ntp.org
FallbackNTP=time.google.com time.cloudflare.com
```

```bash
systemctl restart systemd-timesyncd
```

## Using a Kubernetes DaemonSet for NTP

If nodes don't have network access to public NTP servers, run an NTP server inside the cluster:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ntp-sync
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: ntp-sync
  template:
    metadata:
      labels:
        app: ntp-sync
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: ntp
        image: cturra/ntp:latest
        securityContext:
          privileged: true
```

## Adjusting Ceph's Clock Tolerance (Last Resort)

If synchronization is imperfect due to high-latency networks, increase the tolerance:

```bash
ceph config set mon mon_clock_drift_allowed 0.1
ceph config set mon mon_clock_drift_warn_backoff 30
```

## Verifying the Fix

After fixing NTP sync, check that the warning clears:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health
```

The output should return to `HEALTH_OK` or remove the clock skew warning.

## Summary

Ceph clock skew errors result from time drift between monitor nodes exceeding 0.05 seconds. The fix involves ensuring chrony or systemd-timesyncd is running and synchronized on all monitor nodes. For air-gapped environments, deploy an internal NTP server accessible to all Kubernetes nodes. Adjusting Ceph's tolerance is a last resort for high-latency network environments.
