# How to Set osd_recovery_op_priority in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Recovery, Priority, OSD, Storage

Description: Learn how to set osd_recovery_op_priority in Ceph to control how recovery operations compete with client I/O for OSD resources in Rook-managed clusters.

---

## What Is osd_recovery_op_priority?

`osd_recovery_op_priority` controls the I/O priority of recovery operations relative to client I/O. Ceph uses a priority queue for OSD operations, and this parameter determines where recovery ops are placed in that queue.

The scale ranges from 1 (lowest priority) to 63 (highest priority). The companion parameter `osd_client_op_priority` defaults to 63, meaning client I/O has maximum priority by default, while recovery at its default of 3 is nearly 21 times lower priority.

## Checking Current Priority

```bash
ceph config get osd osd_recovery_op_priority
ceph config get osd osd_client_op_priority
```

Show all OSD config overrides:

```bash
ceph config dump | grep -i priority
```

## Setting osd_recovery_op_priority

### Increase recovery priority during maintenance

```bash
ceph config set osd osd_recovery_op_priority 15
```

### Reset to default after recovery

```bash
ceph config set osd osd_recovery_op_priority 3
```

### Apply immediately without restart

```bash
ceph tell osd.* injectargs '--osd-recovery-op-priority=15'
```

Verify the change was applied:

```bash
ceph config get osd osd_recovery_op_priority
```

## Priority Tiers - Practical Guidelines

| Scenario | Recovery Priority | Client Priority |
|----------|------------------|----------------|
| Normal production | 3 | 63 |
| Low-traffic period | 10 | 63 |
| Maintenance window | 30 | 63 |
| Emergency recovery (no clients) | 63 | 63 |

## Per-Pool Priority Override

You can also set recovery priority at the pool level for pools with critical data:

```bash
ceph osd pool set <pool-name> recovery_priority 10
```

List pools with custom recovery priorities:

```bash
ceph osd dump | grep recovery_priority
```

## Combined with Other Recovery Throttles

`osd_recovery_op_priority` works alongside other throttling parameters. For balanced recovery with minimal client impact, combine settings:

```bash
ceph config set osd osd_recovery_op_priority 5
ceph config set osd osd_recovery_max_active 2
ceph config set osd osd_recovery_sleep_hdd 0.3
```

## Rook Configuration

Manage recovery op priority persistently via CephCluster:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephConfig:
    osd:
      osd_recovery_op_priority: "5"
      osd_client_op_priority: "63"
```

For runtime adjustment via the toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_recovery_op_priority 10
```

## Monitoring the Effect on Latency

Observe client write and read latency after changing priority:

```bash
ceph osd perf
```

Target latency values vary by storage tier:
- NVMe: under 1ms acceptable
- SSD: under 5ms acceptable
- HDD: under 20ms acceptable

## Summary

`osd_recovery_op_priority` determines how aggressively Ceph recovery competes with client I/O. The default of 3 (versus client priority 63) strongly favors clients. Raising this value speeds recovery but increases application latency. Scheduling higher priority during off-peak hours or maintenance windows provides the best of both outcomes.
