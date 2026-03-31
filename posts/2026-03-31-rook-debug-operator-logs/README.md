# How to Debug Rook Operator Logs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Operator, Logging, Debugging

Description: Learn how to read and interpret Rook-Ceph operator logs to debug cluster provisioning failures, reconciliation errors, and CRD processing issues.

---

## Overview

The Rook-Ceph operator is the central controller that reconciles all Ceph CRDs. When a cluster is not behaving as expected - OSDs not created, monitors failing, pools not provisioned - the operator logs are the primary diagnostic resource. Understanding how to navigate and interpret them makes troubleshooting significantly faster.

## Accessing Operator Logs

The operator runs as a single deployment (or two replicas with leader election):

```bash
kubectl logs -n rook-ceph \
  -l app=rook-ceph-operator \
  --tail=200
```

Follow in real time:

```bash
kubectl logs -n rook-ceph \
  -l app=rook-ceph-operator \
  -f
```

## Enabling Debug Logging

For detailed trace-level output, temporarily increase log verbosity:

```bash
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --set logLevel=DEBUG
```

Or patch the operator deployment directly for immediate effect:

```bash
kubectl set env deployment/rook-ceph-operator \
  -n rook-ceph \
  ROOK_LOG_LEVEL=DEBUG
```

## Key Log Patterns to Search For

### Reconciliation Start/End

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-operator | \
  grep -i "reconcil"
```

Successful reconciliation shows `reconciliation succeeded`. Failures show the error.

### OSD Provisioning

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-operator | \
  grep -i "osd\|device\|disk" | tail -50
```

### Monitor Operations

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-operator | \
  grep -iE "mon|monitor" | tail -50
```

### CRD Processing

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-operator | \
  grep -iE "ceph.*pool|filesystem|objectstore" | tail -30
```

## Interpreting Common Log Messages

**OSD not found on device:**
```text
skipping device "sdb": no valid OSD found or the partition table is not GPT
```
This means the disk has existing partitions or a filesystem. Wipe the device.

**Monitor quorum lost:**
```text
failed to get mon quorum, retrying: ...
```
One or more monitor pods are not running. Check `kubectl get pods -n rook-ceph -l app=rook-ceph-mon`.

**Controller rate limiting:**
```text
Reconciler rate limited, wait time: 10s
```
Expected during rapid state changes. If persistent, the operator may be in a retry loop due to a persistent error.

## Correlating Operator Logs with Events

Cross-reference operator logs with Kubernetes events:

```bash
kubectl get events -n rook-ceph \
  --sort-by='.lastTimestamp' | tail -20
```

## Saving Operator Logs for Support

```bash
kubectl logs -n rook-ceph \
  -l app=rook-ceph-operator \
  --since=1h > rook-operator-$(date +%Y%m%d-%H%M).log
```

## Restoring Normal Log Level

After debugging, restore INFO level to reduce log volume:

```bash
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --set logLevel=INFO
```

## Summary

Rook operator logs are the first stop when diagnosing cluster issues. Enable DEBUG level during active troubleshooting, search for reconciliation errors and OSD/monitor keywords, and correlate with Kubernetes events for a complete picture. Return to INFO level after the issue is resolved to prevent log storage overhead.
