# How to Monitor Actor Placement Distribution in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Placement, Monitoring, Prometheus

Description: Monitor Dapr actor placement distribution across pods using metrics and the placement service API to detect uneven load distribution and rebalancing events.

---

## Overview

Dapr's placement service distributes actors across pods using consistent hashing. Monitoring placement distribution reveals hot spots, uneven load, and helps you size your actor service correctly.

## How Dapr Actor Placement Works

The placement service uses a consistent hash ring to assign actor types to specific pods. Each actor ID maps deterministically to a pod, and when pods are added or removed, a portion of actors are reassigned (rebalanced).

## Enabling Placement Metrics

The placement service exposes metrics on port 9091 by default:

```bash
# Check placement service metrics
kubectl port-forward svc/dapr-placement-server 9091:9091 -n dapr-system

curl http://localhost:9091/metrics | grep dapr_placement
```

Key metrics:

```bash
# Total actors per pod
dapr_placement_actor_count

# Rebalancing events
dapr_placement_actor_rebalanced_total

# Placement latency
dapr_placement_lookup_latency_ms_bucket

# Connected Dapr sidecars
dapr_placement_runtime_total
```

## Prometheus Queries for Placement Health

```bash
# Actor count per pod (should be roughly equal)
dapr_placement_actor_count by (pod)

# Actor distribution coefficient of variation (lower = more even)
stddev(dapr_placement_actor_count) / avg(dapr_placement_actor_count)

# Rebalancing rate (should be low outside deployments)
rate(dapr_placement_actor_rebalanced_total[5m])

# Active actors per actor type
sum by (actor_type) (dapr_actor_activated_total - dapr_actor_deactivated_total)
```

## Grafana Dashboard for Placement

```json
{
  "title": "Actor Placement Distribution",
  "panels": [
    {
      "title": "Actors per Pod",
      "type": "bar",
      "targets": [{
        "expr": "dapr_placement_actor_count"
      }],
      "fieldConfig": {
        "thresholds": {
          "steps": [
            {"color": "green", "value": 0},
            {"color": "yellow", "value": 1000},
            {"color": "red", "value": 2000}
          ]
        }
      }
    },
    {
      "title": "Rebalancing Events/min",
      "type": "stat",
      "targets": [{
        "expr": "sum(rate(dapr_placement_actor_rebalanced_total[1m])) * 60"
      }]
    }
  ]
}
```

## Detecting Hot Spots

```bash
# Alert when one pod has more than 2x the average actor count
- alert: ActorHotSpot
  expr: |
    max(dapr_placement_actor_count) >
    2 * avg(dapr_placement_actor_count)
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Actor placement is uneven - possible hot spot"
```

## Actor Type Distribution

Check per-type counts to find which actor types are concentrated:

```bash
# Activate count per type
sum by (actor_type) (
  rate(dapr_actor_activated_total[5m])
)
```

## Inspecting Placement via Dapr Debug API

```bash
# Get current placement table for your app
curl http://localhost:3500/v1.0/metadata | jq '.actors'

# Response shows registered actor types
{
  "actors": [
    {"type": "OrderActor", "count": 142},
    {"type": "CustomerActor", "count": 87}
  ]
}
```

## Scaling Recommendations

| Actors per Pod | Action |
|---------------|--------|
| < 500 | Healthy |
| 500-2000 | Monitor state store load |
| > 2000 | Scale out actor service replicas |
| Skewed > 2x | Investigate actor ID distribution |

## Summary

Monitor Dapr actor placement distribution using the placement service Prometheus metrics. Track actor count per pod to detect hot spots, watch rebalancing rate to understand deployment impact, and use the Dapr metadata API for a quick view of active actor counts by type. Alert on distribution skew ratios exceeding 2x to catch placement issues before they cause performance degradation.
