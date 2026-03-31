# How to Understand the stale PG State in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, State, Troubleshooting, Storage

Description: Understand the Ceph stale PG state, what causes PGs to go stale, how to diagnose the root cause, and how to recover affected placement groups.

---

A `stale` PG is one where the monitor has not received a status update from the primary OSD within the expected time window. It indicates the primary OSD may be down or unreachable, and the PG's current state is unknown.

## What stale Means

The monitor tracks PG states via heartbeat reports from primary OSDs. If a primary OSD stops reporting, the monitor marks its PGs as `stale` after `mon_osd_report_timeout` seconds (default 900 seconds). Stale PGs may be:

- Temporarily inaccessible (OSD is down)
- Permanently lost (disk failure)
- In a network partition

## Checking Stale PGs

```bash
ceph status
# HEALTH_WARN: X pgs stale

ceph pg stat | grep stale

# List stale PGs with their primary OSD
ceph pg dump | grep stale
```

For detailed info about a stale PG:

```bash
ceph pg <pg-id> query
```

## Why PGs Go Stale

Primary causes:

1. The primary OSD crashed or was powered off
2. Network partition isolates the primary from monitors
3. An OSD process is alive but not responding to Ceph requests
4. All OSDs in the acting set are down

Identify which OSDs host the stale PGs:

```bash
ceph pg dump --format json | jq '.pg_stats[] | select(.state | contains("stale")) | {pgid, acting}'
```

Then check those OSD states:

```bash
ceph osd tree | grep "down\|out"
```

## Recovering from Stale PGs

### If the primary OSD is temporarily down

Start the OSD and it will re-report, removing the stale state:

```bash
systemctl start ceph-osd@<id>.service
watch ceph pg stat
```

### If the primary OSD is permanently lost

Mark the OSD out and let Ceph elect a new primary from the remaining replicas:

```bash
ceph osd out osd.<id>
ceph osd down osd.<id>
```

Ceph will remap the PG to the remaining healthy OSDs and clear the stale state.

## Debugging Stale PGs

```bash
# Check how long PG has been stale
ceph pg <pg-id> query | jq '.info.stats.last_active'

# Check the OSD's last seen time
ceph osd stat

# Check OSD log
journalctl -u ceph-osd@<id> --since "1 hour ago"
```

## Stale PG vs Inactive PG

| State | I/O status | Meaning |
|-------|-----------|---------|
| stale | Unknown | Primary OSD not reporting |
| inactive | Blocked | PG cannot peer - no quorum |
| active+clean | Normal | Fully healthy |

## Preventing Stale PGs

```bash
# Reduce stale timeout (not recommended for busy clusters)
ceph config set mon mon_osd_report_timeout 600

# Ensure OSDs have stable network connectivity
ping -c 10 osd-node1
```

## Summary

Stale PGs indicate that the primary OSD has stopped reporting status to the monitors. They are usually caused by OSD or node failures and resolve automatically when the OSD restarts or when Ceph elects a new primary from the remaining replicas. Stale PGs are a warning, not necessarily a failure, but they require prompt investigation to prevent further degradation.
