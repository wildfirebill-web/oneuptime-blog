# How to Use the Ceph Insights Module

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitoring, Insights, Telemetry

Description: Enable and use the Ceph Insights module to collect cluster health reports, track configuration history, and monitor long-term cluster trends.

---

## What Is the Ceph Insights Module

The Ceph `insights` module is a manager plugin that stores periodic cluster health reports in a RADOS object. It maintains a rolling history of cluster health status, warnings, and errors over time. This allows you to answer questions like "when did this warning first appear?" or "how long has this health issue persisted?"

## Enabling the Insights Module

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mgr module enable insights
```

Verify it is enabled:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mgr module ls | grep insights
```

Output:

```text
    "enabled_modules": [
        "balancer",
        "insights",
        "pg_autoscaler",
        ...
    ]
```

## Viewing the Insights Report

Get the current insights report:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph insights
```

This returns a JSON report of cluster health history. For human-readable output:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph insights | python3 -m json.tool | head -100
```

Example output structure:

```json
{
    "version": 1,
    "osd_stats_history": [...],
    "health": {
        "2026-03-31T10:00:00.000000Z": {
            "checks": {
                "OSD_NEARFULL": {
                    "severity": "HEALTH_WARN",
                    "summary": {
                        "message": "1 nearfull osd(s)"
                    }
                }
            },
            "status": "HEALTH_WARN"
        }
    }
}
```

## Filtering Historical Health Data

The insights module records health at regular intervals. To see when a specific issue first appeared:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph insights | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
for ts, health in sorted(data.get('health', {}).items()):
    status = health.get('status', 'unknown')
    checks = list(health.get('checks', {}).keys())
    if checks:
        print(f'{ts}: {status} - {checks}')
"
```

Output:

```text
2026-03-31T09:00:00: HEALTH_WARN - ['OSD_NEARFULL']
2026-03-31T09:30:00: HEALTH_WARN - ['OSD_NEARFULL', 'SLOW_OPS']
2026-03-31T10:00:00: HEALTH_OK - []
```

## Configuring Retention Period

By default, insights keeps data for several days. Configure the retention period:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set mgr mgr/insights/retention_period 604800
```

This sets 7 days of retention (in seconds).

Check current retention:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config get mgr mgr/insights/retention_period
```

## Pruning Old Insights Data

If the insights object grows large, prune old data:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph insights prune-health 86400
```

This prunes health records older than 86400 seconds (1 day).

## Using Insights for Capacity Planning

The insights module also tracks OSD statistics over time. Export and analyze trends:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph insights > /tmp/insights.json
```

Process with Python to extract OSD usage over time:

```python
import json

with open('/tmp/insights.json') as f:
    data = json.load(f)

for ts, stats in sorted(data.get('osd_stats_history', {}).items()):
    total = sum(s.get('kb', 0) for s in stats.values())
    used = sum(s.get('kb_used', 0) for s in stats.values())
    pct = (used / total * 100) if total else 0
    print(f"{ts}: {pct:.1f}% used")
```

## Disabling the Insights Module

If you don't need it:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mgr module disable insights
```

## Summary

The Ceph Insights module provides historical cluster health tracking without requiring external tools. Enable it with `ceph mgr module enable insights`, then use `ceph insights` to retrieve a JSON history of health states and OSD statistics. This is valuable for post-incident analysis, capacity planning, and understanding the timeline of cluster issues.
