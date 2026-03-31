# How to Use Dapr Actor Timers for In-Memory Scheduled Callbacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Timer, Scheduling, Microservice

Description: Learn how to register and manage Dapr actor timers for lightweight in-memory scheduled callbacks that run while an actor remains active.

---

## Overview

Dapr actor timers allow you to schedule periodic callbacks on an active actor instance. Unlike actor reminders, timers are not persisted and do not fire if the actor is deactivated or the application restarts. Use timers for lightweight, non-critical periodic work such as cache invalidation, heartbeats, or polling.

## When to Use Timers vs Reminders

Use a timer when the scheduled task only matters while the actor is alive. Use a reminder when the task must fire even after restarts or deactivation. A common pattern is using a reminder to ensure the actor stays active, while the timer handles frequent internal work.

## Registering a Timer

```csharp
// .NET SDK - register a timer inside an actor method
await this.RegisterTimerAsync(
    "poll-status",
    nameof(this.PollExternalStatus),
    Encoding.UTF8.GetBytes("context-data"),
    dueTime: TimeSpan.FromSeconds(10),   // first fire after 10 seconds
    period: TimeSpan.FromSeconds(30)     // repeat every 30 seconds
);
```

```python
# Python SDK
await self.register_timer(
    "poll-status",
    self.poll_external_status,
    b"context-data",
    timedelta(seconds=10),
    timedelta(seconds=30)
)
```

## Implementing the Timer Callback

```csharp
public async Task PollExternalStatus(byte[] state)
{
    var contextData = Encoding.UTF8.GetString(state);
    var status = await _externalClient.GetStatusAsync(this.Id.GetId());

    await this.StateManager.SetStateAsync("last-status", status);

    if (status == "completed")
    {
        // Cancel the timer once the work is done
        await this.UnregisterTimerAsync("poll-status");
    }
}
```

## Unregistering a Timer

```python
# Python SDK - cancel the timer explicitly
async def stop_polling(self):
    await self.unregister_timer("poll-status")
```

## Timer Lifecycle Considerations

Timers fire using turn-based concurrency, meaning the actor processes one request at a time. If a timer fires while the actor is processing another request, the timer callback is queued and fires after the current request completes.

```bash
# Observe timer activity in sidecar logs
kubectl logs -l app=actor-service -c daprd | grep -i "timer"
```

## One-Time Timers

Set `period` to `TimeSpan.FromMilliseconds(-1)` in .NET (or equivalent) to create a one-shot timer:

```csharp
await this.RegisterTimerAsync(
    "one-time-action",
    nameof(this.PerformOneTimeAction),
    null,
    dueTime: TimeSpan.FromMinutes(1),
    period: TimeSpan.FromMilliseconds(-1)  // fire once, then stop
);
```

## Summary

Dapr actor timers provide a simple mechanism for scheduling in-memory periodic callbacks on active actor instances. They are lightweight and easy to use but do not survive actor deactivation or application restarts. For critical scheduling needs, use actor reminders instead. Combine timers and reminders to build actors that stay active via reminders and perform frequent work via timers.
