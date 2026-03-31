# How to Calculate Recovery Point Objective (RPO) for MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Backup, Recovery, Reliability, Database

Description: Learn how to calculate and achieve your Recovery Point Objective for MongoDB by analyzing backup frequency, oplog retention, and data loss tolerance.

---

## What Is RPO in the Context of MongoDB

Recovery Point Objective (RPO) is the maximum amount of data your organization can afford to lose in a disaster scenario. If your RPO is 1 hour, you must be able to restore MongoDB to a state no more than 1 hour before the failure. RPO drives your backup frequency and replication strategy.

RPO is expressed as time, not data volume. Losing 1 hour of transactions means losing everything written in that window - even if the write volume is tiny.

## Calculating Your MongoDB RPO

Your effective RPO is determined by the gap between your most recent valid, restorable backup and the point of failure:

```text
RPO = Time of Failure - Timestamp of Last Valid Backup
```

For example, if your nightly backup completes at 02:00 and a failure occurs at 14:00, your actual RPO exposure is 12 hours - regardless of what your SLA says.

To achieve a target RPO, your backup frequency must be at most equal to that RPO:

```python
def calculate_backup_frequency(target_rpo_hours, restore_time_hours=1.0):
    """
    Calculate required backup frequency given an RPO target.
    restore_time accounts for the time needed to actually perform the restore.
    """
    effective_window = target_rpo_hours - restore_time_hours
    if effective_window <= 0:
        raise ValueError("RPO too short - restore time exceeds RPO target")

    print(f"Target RPO: {target_rpo_hours}h")
    print(f"Estimated restore time: {restore_time_hours}h")
    print(f"Required backup interval: <= {effective_window}h")
    print(f"Minimum backup frequency: every {effective_window * 60:.0f} minutes")

# Example calculations
calculate_backup_frequency(target_rpo_hours=4.0, restore_time_hours=0.5)
# Required backup interval: <= 3.5h
# Minimum backup frequency: every 210 minutes
```

## Measuring Your Current RPO Gap

Query your backup history to understand your actual RPO:

```javascript
// Assuming you track backups in a MongoDB collection
db.backup_history.aggregate([
  {$match: {status: "success"}},
  {$sort: {completed_at: 1}},
  {$setWindowFields: {
    sortBy: {completed_at: 1},
    output: {
      prev_completed: {
        $shift: {output: "$completed_at", by: -1}
      }
    }
  }},
  {$project: {
    completed_at: 1,
    gap_minutes: {
      $divide: [
        {$subtract: ["$completed_at", "$prev_completed"]},
        60000
      ]
    }
  }},
  {$group: {
    _id: null,
    avg_gap_min: {$avg: "$gap_minutes"},
    max_gap_min: {$max: "$gap_minutes"}
  }}
])
```

## Achieving Sub-Hourly RPO with Oplog Replay

For RPO targets under 1 hour, point-in-time recovery via continuous oplog capture is required:

```bash
# Check current oplog window (how far back you can restore)
mongosh --eval "
  const first = db.getSiblingDB('local').oplog.rs.find().sort({'\$natural':1}).limit(1)[0];
  const last = db.getSiblingDB('local').oplog.rs.find().sort({'\$natural':-1}).limit(1)[0];
  const windowHours = (last.ts.t - first.ts.t) / 3600;
  print('Oplog window: ' + windowHours.toFixed(2) + ' hours');
  print('This is your theoretical minimum RPO with continuous oplog backup');
"
```

If the oplog window is smaller than your RPO, increase the oplog size:

```bash
# Increase oplog to 10GB
mongosh --eval "db.adminCommand({replSetResizeOplog: 1, size: 10240})"
```

## RPO for Different MongoDB Deployment Types

| Deployment | Mechanism | Achievable RPO |
|---|---|---|
| Standalone | mongodump + cron | Backup interval |
| Replica Set | Oplog capture | Seconds to minutes |
| Atlas M10+ | Continuous backup (PITR) | 1 second |
| Sharded Cluster | PBM incremental | Minutes |

## Summary

MongoDB RPO is the maximum tolerable data loss window, calculated as the gap between the last valid backup and the point of failure. Achieving a low RPO requires frequent backups and, for sub-hour targets, continuous oplog capture or MongoDB Atlas PITR. Measure your current backup gap by tracking backup history in a collection and reviewing max gap intervals. Monitor oplog window size to ensure your replication log covers your RPO window at all times.
