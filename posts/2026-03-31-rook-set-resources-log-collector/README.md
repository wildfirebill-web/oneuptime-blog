# How to Set Resources for Rook-Ceph Log Collector

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Log Collector, Resources, Pod, Logging

Description: Configure resource requests and limits for Rook-Ceph log collector pods to ensure Ceph daemon logs are collected reliably and efficiently across all cluster nodes.

---

## Overview

The Rook-Ceph log collector gathers logs from Ceph daemons running on each Kubernetes node and makes them accessible via standard `kubectl logs`. It runs as a sidecar or DaemonSet component depending on Rook configuration. Proper resource allocation ensures log collection keeps up with high-volume logging without impacting Ceph daemon performance.

## Configuring Log Collector Resources

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  logCollector:
    enabled: true
    periodicity: "daily"
    maxLogSize: "500M"
  resources:
    logcollector:
      requests:
        cpu: "100m"
        memory: "100Mi"
      limits:
        cpu: "500m"
        memory: "1Gi"
```

Apply and check:

```bash
kubectl apply -f cephcluster.yaml

# Verify log collector pods
kubectl -n rook-ceph get pods | grep log

# Check resource settings
kubectl -n rook-ceph describe pod rook-ceph-logcollector-<node>-<hash> | \
    grep -A10 "Limits:"
```

## Log Collector Configuration Options

```yaml
spec:
  logCollector:
    enabled: true
    # How often to rotate collected logs
    periodicity: "daily"   # Options: hourly, daily, weekly
    # Maximum size before rotation
    maxLogSize: "500M"
```

## Viewing Collected Logs

```bash
# View logs from a specific log collector pod
kubectl -n rook-ceph logs rook-ceph-logcollector-<node>-<hash>

# Follow logs in real time
kubectl -n rook-ceph logs -f rook-ceph-logcollector-<node>-<hash>

# Check log sizes on node
kubectl -n rook-ceph exec rook-ceph-logcollector-<node>-<hash> -- \
    ls -lh /var/log/ceph/
```

## Resource Impact of High Log Verbosity

When debugging is enabled, log volume increases dramatically:

```bash
# Check current debug log levels
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
    ceph config get osd debug_osd

# High verbosity (debug level 20) can generate 100+ MB/min
# Increase log collector memory limit temporarily during debug sessions
```

```yaml
# Temporary high-verbosity resource override
spec:
  resources:
    logcollector:
      requests:
        cpu: "200m"
        memory: "512Mi"
      limits:
        cpu: "1000m"
        memory: "2Gi"
```

## Log Rotation and Disk Space

```bash
# Check disk usage by log collector on a node
kubectl -n rook-ceph exec rook-ceph-logcollector-<node>-<hash> -- \
    du -sh /var/log/ceph/*

# Force log rotation if disk space is low
kubectl -n rook-ceph exec rook-ceph-logcollector-<node>-<hash> -- \
    logrotate -f /etc/logrotate.d/ceph
```

## Integrating with External Log Systems

For production, forward logs to a central system:

```yaml
# Fluent Bit DaemonSet to forward Ceph logs to Elasticsearch
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-ceph
  namespace: rook-ceph
data:
  fluent-bit.conf: |
    [INPUT]
        Name   tail
        Path   /var/log/ceph/*.log
        Tag    ceph.*

    [OUTPUT]
        Name   es
        Match  ceph.*
        Host   elasticsearch.logging.svc
        Port   9200
        Index  ceph-logs
```

## Sizing Guidelines

| Cluster Debug Level | CPU Request | Memory Limit | Notes |
|---|---|---|---|
| Normal (level 1) | 100m | 256Mi | Low log volume |
| Moderate (level 5) | 200m | 512Mi | Elevated during tuning |
| Debug (level 10+) | 500m | 1Gi | Short-term debugging only |
| Max debug (level 20) | 1000m | 2Gi | Emergency diagnosis only |

## Summary

Rook-Ceph log collector pods are lightweight under normal conditions but can require more resources during debug logging sessions. Configure the base memory limit at 256-512Mi and increase temporarily when enabling verbose logging. Set the `maxLogSize` and `periodicity` options to prevent runaway disk consumption, and consider forwarding logs to an external system for long-term retention and analysis.
