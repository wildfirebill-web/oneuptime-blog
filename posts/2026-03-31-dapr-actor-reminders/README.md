# How to Use Dapr Actor Reminders for Persistent Scheduled Tasks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Reminder, Scheduling, Microservice

Description: Learn how to use Dapr actor reminders to schedule persistent recurring or one-time callbacks that survive actor deactivation and application restarts.

---

## Overview

Dapr actor reminders are durable scheduled callbacks associated with a specific actor instance. Unlike actor timers, reminders persist to the actor state store and fire even if the actor is deactivated or the application restarts, making them ideal for business-critical scheduled tasks.

## Reminders vs Timers

| Feature | Reminder | Timer |
|---|---|---|
| Persisted | Yes | No |
| Survives restart | Yes | No |
| Survives deactivation | Yes | No |
| Use case | Critical scheduling | In-memory polling |

## Registering a Reminder

```python
# Python SDK - register a reminder from inside an actor method
async def start_subscription_renewal(self):
    await self.register_reminder(
        "renew-subscription",        # reminder name
        b"renewal-data",             # state passed to callback
        timedelta(days=30),          # due time (first fire)
        timedelta(days=30)           # period (repeat interval)
    )
```

```csharp
// .NET SDK - register a one-time reminder
await this.RegisterReminderAsync(
    "send-welcome-email",
    Encoding.UTF8.GetBytes(userId),
    dueTime: TimeSpan.FromMinutes(5),
    period: TimeSpan.FromMilliseconds(-1)  // -1 means fire once
);
```

## Implementing the Reminder Callback

Your actor must implement the `IRemindable` interface (or equivalent in your SDK) to receive reminder callbacks:

```csharp
public class SubscriptionActor : Actor, ISubscriptionActor, IRemindable
{
    public async Task ReceiveReminderAsync(
        string reminderName,
        byte[] state,
        TimeSpan dueTime,
        TimeSpan period)
    {
        if (reminderName == "renew-subscription")
        {
            var userId = Encoding.UTF8.GetString(state);
            await RenewSubscriptionAsync(userId);
        }
    }
}
```

## Unregistering a Reminder

```python
# Python SDK - unregister when the task is no longer needed
async def cancel_renewal(self):
    await self.unregister_reminder("renew-subscription")
```

## Listing Reminders via the API

```bash
# Get all reminders for a specific actor
curl http://localhost:3500/v1.0/actors/SubscriptionActor/user-42/reminders/renew-subscription
```

## Observing Reminder Execution

```bash
# Monitor reminder-related logs
kubectl logs -l app=actor-service -c daprd | grep -i "reminder"
```

## Summary

Dapr actor reminders provide durable, persistent scheduling tied to specific actor instances. They survive actor deactivation and application restarts by persisting to the configured state store. Use reminders for critical business scheduling tasks like subscription renewals, follow-up notifications, or deadline enforcement, and use actor timers for lightweight in-memory polling that does not need to survive restarts.
