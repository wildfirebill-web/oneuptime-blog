# How to Use the Insights Module in Ceph Manager

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Insights, Monitoring, Health

Description: Learn how to use the Ceph Manager Insights module to collect periodic cluster health reports for long-term trend analysis and incident review.

---

The Ceph Manager Insights module collects periodic health and performance snapshots of your Ceph cluster. These snapshots serve as a historical record, helping operators identify trends and understand what was happening before and during incidents.

## Enabling the Insights Module

Enable the module with:

```bash
ceph mgr module enable insights
```

The module immediately begins recording health reports at regular intervals.

## Viewing the Report

Generate a formatted insights report showing recent cluster health history:

```bash
ceph insights
```

Example output includes timestamped health check summaries:

```json
{
  "version": 1,
  "health": {
    "2026-03-31 09:00:00": {
      "status": "HEALTH_OK",
      "checks": {}
    },
    "2026-03-31 08:00:00": {
      "status": "HEALTH_WARN",
      "checks": {
        "OSD_DOWN": {
          "severity": "HEALTH_WARN",
          "summary": {"message": "1 osds down"}
        }
      }
    }
  }
}
```

## Filtering by Time Range

Limit the report to a specific time window:

```bash
ceph insights --since "2026-03-31 06:00:00"
```

## Clearing Historical Data

Remove accumulated insights history:

```bash
ceph insights prune-health
```

This is useful after major cluster changes to start a fresh baseline.

## Configuring the Collection Interval

The insights module stores health data hourly by default. Each data point represents a one-hour bucket of cluster health. Configure how much history to retain:

```bash
ceph config set mgr mgr/insights/max_health_period 604800
```

This sets the maximum retention to 7 days (604800 seconds).

## Use Case: Post-Incident Review

When an OSD failure occurs, use insights to check what health warnings preceded it:

```bash
ceph insights --since "2026-03-31 00:00:00" | python3 -m json.tool | grep -A5 "HEALTH_WARN"
```

This surfaces any early warning signs such as slow requests, clock skew, or PG degradation that occurred before the failure.

## Combining with the Crash Module

Pair insights with crash reports for a complete incident timeline:

```bash
# Get the time of a crash
ceph crash ls

# Review cluster health at that time using insights
ceph insights --since "2026-03-31 08:00:00"
```

## Summary

The Ceph Manager Insights module captures periodic health snapshots of your cluster, building a historical record useful for trend analysis and incident post-mortems. Enabling it requires a single command, and the `ceph insights` command retrieves a time-ranged view of cluster health changes, helping operators understand the sequence of events leading to any cluster issue.
