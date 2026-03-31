# How to Monitor Dapr Actor Activation Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Metric, Observability, Prometheus

Description: Track Dapr actor activation, deactivation, and active count metrics to understand actor lifecycle behavior and resource usage patterns.

---

Dapr actors have a unique lifecycle - they activate on first call and deactivate after an idle timeout. Monitoring activation metrics helps you understand how many actors are running, how frequently they cycle, and whether your placement service is distributing actors evenly.

## Key Actor Metrics

Dapr exposes these actor lifecycle metrics:

- `dapr_actor_active_actors` - current count of active actors by type
- `dapr_actor_activated_total` - cumulative activations
- `dapr_actor_deactivated_total` - cumulative deactivations
- `dapr_actor_pending_actor_calls` - queued calls waiting for an actor
- `dapr_actor_timers_fired_total` - timer executions
- `dapr_actor_reminders_fired_total` - reminder executions

## Querying Active Actor Count

```
# Active actors per type
dapr_actor_active_actors{app_id="order-processor"}

# Total active actors across all types
sum(dapr_actor_active_actors)

# Active actors per type, per node
dapr_actor_active_actors
```

## Activation and Deactivation Rate

```
# Activation rate (new actors being created)
rate(dapr_actor_activated_total[5m])

# Deactivation rate (actors going idle)
rate(dapr_actor_deactivated_total[5m])

# Net change rate (activations minus deactivations)
rate(dapr_actor_activated_total[5m]) - rate(dapr_actor_deactivated_total[5m])
```

High deactivation rates mean your actors are frequently going idle, which increases activation overhead. Consider adjusting the idle timeout if this causes latency spikes.

## Pending Call Queue Depth

A growing pending call queue indicates actors cannot keep up with demand:

```
# Pending calls per actor type
dapr_actor_pending_actor_calls{actor_type="OrderActor"}

# Alert if queue is building up
dapr_actor_pending_actor_calls > 100
```

## Timer and Reminder Activity

```
# Timer firing rate per actor type
rate(dapr_actor_timers_fired_total[5m])

# Reminder firing rate
rate(dapr_actor_reminders_fired_total[5m])

# Reminder failures
rate(dapr_actor_reminder_fired_failed_total[5m])
```

## Alert Rules for Actor Metrics

```yaml
groups:
- name: dapr-actors
  rules:
  - alert: DaprActorHighPendingCalls
    expr: dapr_actor_pending_actor_calls > 50
    for: 3m
    labels:
      severity: warning
    annotations:
      summary: "Actor type {{ $labels.actor_type }} has {{ $value }} pending calls"

  - alert: DaprActorReminderFailures
    expr: rate(dapr_actor_reminder_fired_failed_total[5m]) > 0
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Actor reminders failing for type {{ $labels.actor_type }}"

  - alert: DaprActorCountAnomaly
    expr: |
      abs(
        dapr_actor_active_actors
        - avg_over_time(dapr_actor_active_actors[30m])
      ) > avg_over_time(dapr_actor_active_actors[30m]) * 0.5
    for: 10m
    labels:
      severity: info
    annotations:
      summary: "Unusual change in active actor count for {{ $labels.actor_type }}"
```

## Grafana Visualization

Create a dashboard panel showing actor lifecycle over time:

```
# Panel 1 - Active actors over time
dapr_actor_active_actors{app_id="$app_id"}

# Panel 2 - Activation/deactivation rates
rate(dapr_actor_activated_total{app_id="$app_id"}[5m])
rate(dapr_actor_deactivated_total{app_id="$app_id"}[5m])

# Panel 3 - Pending calls heatmap
dapr_actor_pending_actor_calls{app_id="$app_id"}
```

## Summary

Actor activation metrics in Dapr reveal the health of your actor-based workflows. Monitor active actor counts to understand memory pressure, watch pending call queues to detect throughput bottlenecks, and track reminder failure rates to ensure scheduled tasks execute reliably. Sudden spikes in activations or deactivations often indicate traffic pattern changes that require adjustment of idle timeouts or actor placement configuration.
