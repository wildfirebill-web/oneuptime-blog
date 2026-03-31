# How to Watch Live Cluster Events with ceph -w

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitoring, Events, Diagnostics

Description: Use ceph -w to stream live cluster events including OSD state changes, PG transitions, and health check updates in your Rook-Ceph cluster.

---

`ceph -w` streams real-time cluster log messages and events to your terminal. It is invaluable for watching what happens during OSD failures, recovery operations, upgrades, or configuration changes.

## Start Live Event Monitoring

```bash
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- ceph -w
```

The command shows the current cluster status, then streams new events as they occur:

```text
  cluster:
    id:     a3f8e12d-1234-5678-abcd-000000000000
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c
    osd: 6 OSDs; 6 up, 6 in

2026-03-31T10:22:01.123456+0000 mon.a [INF] osd.3 marked itself down
2026-03-31T10:22:03.234567+0000 mon.a [WRN] Health check failed: OSD_DOWN
2026-03-31T10:22:05.345678+0000 mon.a [INF] osd.3 boot
2026-03-31T10:22:06.456789+0000 mon.a [INF] Health check cleared: OSD_DOWN
```

## Event Format

Each event line contains:
- Timestamp (ISO 8601)
- Source daemon (e.g., `mon.a`, `osd.3`, `mds.myfs-a`)
- Severity: `[INF]`, `[WRN]`, or `[ERR]`
- Event message

## Common Event Patterns

### OSD Going Down and Recovering

```text
2026-03-31T10:22:01.123456 mon.a [INF] osd.3 marked itself down
2026-03-31T10:22:01.234567 mon.a [WRN] Health check update: OSD_DOWN (1 osds down)
2026-03-31T10:22:45.345678 mon.a [INF] pgmap recovery_io 256 MiB/s, 64 objects/s
2026-03-31T10:24:22.456789 mon.a [INF] osd.3 boot
2026-03-31T10:24:23.567890 mon.a [INF] Health check cleared: OSD_DOWN
```

### PG State Changes

```text
2026-03-31T10:22:01.123456 mon.a [INF] pgmap v12345: 113 pgs: 112 active+clean, 1 active+recovering
2026-03-31T10:22:45.234567 mon.a [INF] pgmap v12346: 113 pgs: 113 active+clean
```

### Health Transitions

```text
2026-03-31T10:22:01.123456 mon.a [WRN] overall HEALTH_WARN (1 osds down)
2026-03-31T10:24:24.234567 mon.a [INF] overall HEALTH_OK
```

## Filter Events by Severity

`ceph -w` does not have built-in filtering, but you can pipe it through `grep`:

```bash
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph -w | grep -E "\[WRN\]|\[ERR\]"
```

## Watch Events During Specific Operations

Monitor events while draining a node:

```bash
# In terminal 1 - watch events
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- ceph -w

# In terminal 2 - drain the node
kubectl drain node3 --ignore-daemonsets --delete-emptydir-data
```

## Record Events to a File

```bash
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- ceph -w \
  > /tmp/ceph-events-$(date +%Y%m%d).log 2>&1 &
```

## Alternative: View Recent Log Messages

If you missed events and want historical data:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph log last 100
```

Or filter by channel:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph log last 50 warn cluster
```

## Summary

`ceph -w` provides a real-time event stream from the Ceph cluster log, showing OSD state transitions, PG changes, and health check activations with timestamps and severity levels. Use it to monitor operations like node drains, OSD replacements, or upgrades, and pipe through `grep` to focus on warnings and errors only.
