# How to Handle Actor Garbage Collection in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Garbage Collection, Idle Timeout, Memory Management

Description: Learn how Dapr performs actor garbage collection via idle timeout and scan intervals, and how to tune these settings to balance memory usage and reactivation cost.

---

Dapr's virtual actor model creates actor instances on demand and destroys them when idle. Understanding this garbage collection mechanism is critical for managing memory usage and state store load in production.

## How Actor Garbage Collection Works

Dapr periodically scans all active actors on each host to find ones that have exceeded their idle timeout. When an actor is marked for collection:

1. The `OnDeactivate` callback is invoked on the actor instance.
2. Any pending state saves are completed.
3. The actor instance is removed from in-memory state.
4. The placement service is updated to reflect that the actor is no longer active on this host.

The persistent state in the state store is NOT deleted on deactivation - only the in-memory actor instance is removed.

## Configuring Idle Timeout and Scan Interval

```json
{
  "entities": ["Counter", "UserSession"],
  "actorIdleTimeout": "1h",
  "actorScanInterval": "30s"
}
```

- `actorIdleTimeout`: How long since the last method call before the actor is eligible for collection.
- `actorScanInterval`: How often Dapr scans for idle actors on a host.

## Tuning for Different Workloads

Short-lived session actors:

```json
{
  "actorIdleTimeout": "15m",
  "actorScanInterval": "60s"
}
```

Long-running workflow actors:

```json
{
  "actorIdleTimeout": "24h",
  "actorScanInterval": "5m"
}
```

IoT device twins (devices connect infrequently):

```json
{
  "actorIdleTimeout": "72h",
  "actorScanInterval": "30m"
}
```

## Implementing OnDeactivate for Clean Shutdown

```go
func (a *SessionActor) OnDeactivate() error {
  log.Printf("Deactivating session actor %s", a.ID())

  // Flush any buffered writes before deactivation
  var session SessionData
  if err := a.GetStateManager().Get(context.Background(), "session", &session); err == nil {
    session.LastSeen = time.Now().UTC()
    a.GetStateManager().Set(context.Background(), "session", session)
  }

  return nil
}
```

## Monitoring Garbage Collection Activity

Track deactivation rate via Prometheus:

```promql
# Deactivations per minute
sum(rate(dapr_actor_deactivations_total[1m])) by (actor_type)
```

If deactivation rate is zero over a long period while actor count keeps growing, your idle timeout may be too high:

```bash
# Check active actor count
curl http://localhost:9090/metrics | grep dapr_actor_active_actors
```

## State Store Cleanup

Deactivated actors leave their state in the state store. For truly ephemeral actors, clean up state explicitly during deactivation:

```go
func (a *EphemeralActor) OnDeactivate() error {
  // Remove all state keys
  sm := a.GetStateManager()
  sm.Remove(context.Background(), "data")
  sm.Remove(context.Background(), "metadata")
  return nil
}
```

## Best Practices

- Set `actorScanInterval` to at most 10% of `actorIdleTimeout` for timely collection.
- Avoid very short idle timeouts (under 1 minute) for actors with expensive activation logic.
- Monitor the ratio of activations to deactivations to detect accumulation trends.
- Clean up state store keys for truly ephemeral actors to prevent unbounded storage growth.

## Summary

Dapr's actor garbage collection uses idle timeout and periodic scanning to reclaim in-memory resources from inactive actors. Tuning these parameters to match your workload's access patterns balances memory efficiency against reactivation overhead. Implementing a clean `OnDeactivate` hook ensures state is saved correctly and ephemeral data is removed before the actor instance is destroyed.
