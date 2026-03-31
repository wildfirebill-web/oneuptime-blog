# How to Use db.printSecondaryReplicationInfo() in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replication, Secondary, Lag, Replica Set

Description: Learn how db.printSecondaryReplicationInfo() shows replication lag for each secondary in a replica set to help you identify and troubleshoot lagging nodes.

---

## Overview

`db.printSecondaryReplicationInfo()` (also accessible as `rs.printSecondaryReplicationInfo()`) prints replication lag information for every secondary member in a replica set. Run it from the primary to see how far behind each secondary is relative to the current time.

```javascript
db.printSecondaryReplicationInfo()
```

Example output:

```text
source: mongo-secondary-1:27017
  syncedTo: Tue Mar 31 2026 05:58:00 GMT+0000
  0 secs (0 hrs) behind the primary

source: mongo-secondary-2:27017
  syncedTo: Tue Mar 31 2026 05:30:00 GMT+0000
  1680 secs (0.47 hrs) behind the primary
```

## Interpreting the Output

For each secondary, the output shows:

- **source** - The hostname and port of the secondary.
- **syncedTo** - The timestamp of the last operation the secondary has applied from the primary's oplog.
- **behind** - The lag in seconds and hours between the secondary's sync point and the current time on the primary.

A secondary that is falling significantly behind may be under heavy load, experiencing network latency, or struggling with disk I/O.

## Using rs.printSecondaryReplicationInfo()

```javascript
// Preferred alias in newer shell versions
rs.printSecondaryReplicationInfo()
```

Both commands are equivalent and interchangeable.

## Querying Replication Lag Programmatically

For monitoring scripts, use `rs.status()` instead to get structured data you can parse:

```javascript
const status = rs.status();
const now = new Date();

status.members
  .filter(m => m.stateStr === 'SECONDARY')
  .forEach(m => {
    const lagMs = now - m.optimeDate;
    const lagSec = (lagMs / 1000).toFixed(0);
    print(`${m.name}: ${lagSec}s lag`);
  });
```

## Alerting on Excessive Lag

```javascript
const MAX_LAG_SECONDS = 60;
const status = rs.status();
const now = new Date();

status.members
  .filter(m => m.stateStr === 'SECONDARY')
  .forEach(m => {
    const lagSec = (now - m.optimeDate) / 1000;
    if (lagSec > MAX_LAG_SECONDS) {
      print(`ALERT: ${m.name} is ${lagSec.toFixed(0)}s behind the primary`);
    }
  });
```

## Common Causes of High Replication Lag

- **Heavy write workload** - The secondary cannot apply operations as fast as they are written on the primary.
- **Secondary performing intensive reads** - Long-running queries on the secondary can delay oplog application if they hold locks.
- **Network issues** - High latency or packet loss between primary and secondary.
- **Disk I/O saturation** - Slow disks on the secondary cause oplog application to fall behind.

## Tuning Options

If a specific secondary is consistently lagging, consider:

1. Moving the secondary to a host with faster I/O.
2. Reducing read traffic on the secondary.
3. Increasing the `oplogSizeMB` on the primary to allow more time for recovery.
4. Using `priority: 0` and `hidden: true` for analytics secondaries to prevent them from becoming primary.

```javascript
cfg = rs.conf()
cfg.members[2].priority = 0
cfg.members[2].hidden = true
rs.reconfig(cfg)
```

## Summary

`db.printSecondaryReplicationInfo()` gives a quick human-readable summary of replication lag across all secondaries in your replica set. For production monitoring, prefer `rs.status()` for structured data you can parse and alert on. High replication lag is a warning sign that requires investigation into the secondary's load, network, and disk performance before it falls outside the primary's oplog window and requires a full resync.
