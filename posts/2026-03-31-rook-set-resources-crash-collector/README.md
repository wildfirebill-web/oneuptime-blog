# How to Set Resources for Rook-Ceph Crash Collector

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Crash Collector, Resource, Pod, Monitoring

Description: Configure resource requests and limits for the Rook-Ceph crash collector daemon pods to ensure crash reports are collected reliably without impacting cluster performance.

---

## Overview

The Ceph crash collector (`ceph-crash`) collects crash reports from Ceph daemons and submits them to the Ceph crash module for analysis. In Rook, crash collector pods run as a DaemonSet across all nodes hosting Ceph daemons. These pods are lightweight but need appropriate resource configuration to avoid being starved or disrupting other workloads.

## Understanding the Crash Collector

The crash collector:
- Monitors `/var/lib/ceph/crash/` for new crash reports
- Submits reports to the MGR crash module
- Runs as a low-priority background process
- Consumes minimal CPU and memory under normal conditions

It spikes briefly when a crash occurs as it reads and submits the crash report, but this is typically sub-second.

## Configuring Crash Collector Resources

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  resources:
    crashcollector:
      requests:
        cpu: "15m"
        memory: "60Mi"
      limits:
        cpu: "500m"
        memory: "60Mi"
```

Apply and verify:

```bash
kubectl apply -f cephcluster.yaml

# Check crash collector pods are running
kubectl -n rook-ceph get pods -l app=rook-ceph-crashcollector

# Verify resource settings
kubectl -n rook-ceph describe pod rook-ceph-crashcollector-<node>-<hash> | \
    grep -A10 "Limits:"
```

## Verifying Crash Collection is Working

```bash
# Check if any crash reports are pending
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph crash ls

# View crash report details
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph crash info <crash-id>

# Archive old crash reports
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph crash archive-all

# Check crash collector logs
kubectl -n rook-ceph logs -l app=rook-ceph-crashcollector --tail=50
```

## Checking the Crash Module Status

```bash
# View crash stats summary
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph crash stat

# Check for recent crashes (last 24h)
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
    ceph crash ls-new | head -20
```

## When Crash Collector Needs More Resources

Crash collector may need more resources when:
- Running on nodes with very limited resources
- Multiple daemons crash simultaneously creating large report files
- Long-running crash reports contain large core dumps

```yaml
# For nodes with large Ceph daemons (OSD nodes with many OSDs):
spec:
  resources:
    crashcollector:
      requests:
        cpu: "50m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

## Disabling Crash Collector (Not Recommended)

```yaml
# Disable crash collection (loses crash diagnostic data)
spec:
  crashCollector:
    disable: true
```

Only disable if you have resource constraints that cannot be resolved otherwise.

## Checking Node-Level Resource Usage

```bash
# Verify crash collector is not impacting node resources
kubectl -n rook-ceph top pods | grep crashcollector

# Check if crash collector is triggering OOM on any node
kubectl -n rook-ceph get events --field-selector reason=OOMKilling | grep crash
```

## Summary

The Rook-Ceph crash collector is a lightweight DaemonSet that requires minimal resources - typically 15-50m CPU and 60-128Mi memory. Set a generous CPU limit (500m) to handle brief spikes during crash submission, and a fixed memory limit matching the request since the process is relatively flat in memory usage. Verify crash collection is working by periodically checking `ceph crash ls` and archiving old reports to keep the list manageable.
