# How to Implement Timer-Based Polling with Dapr Actors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Timer, Polling, Scheduled Task

Description: Learn how to implement timer-based polling using Dapr actor timers to periodically check external systems and process work items on a per-entity schedule.

---

## Actors vs Cron Jobs for Polling

Kubernetes CronJobs poll at a cluster level. When you need per-entity polling - check this specific subscription, monitor that specific device, poll this customer's account - Dapr actor timers are more appropriate. Each actor schedules its own timer, independent of every other entity.

## Subscription Monitor Actor

```csharp
[Actor(TypeName = "SubscriptionMonitor")]
public class SubscriptionMonitorActor : Actor, ISubscriptionMonitorActor, IRemindable
{
    private const string StateKey = "subscription";
    private const string TimerName = "poll-timer";

    public SubscriptionMonitorActor(ActorHost host) : base(host) { }

    public async Task StartMonitoringAsync(string subscriptionId, int pollIntervalSeconds)
    {
        await StateManager.SetStateAsync(StateKey, new SubscriptionState
        {
            SubscriptionId = subscriptionId,
            PollIntervalSeconds = pollIntervalSeconds,
            LastChecked = DateTime.MinValue
        });

        // Register the recurring timer
        await RegisterTimerAsync(
            TimerName,
            nameof(PollAsync),
            null,
            TimeSpan.FromSeconds(5),                              // initial delay
            TimeSpan.FromSeconds(pollIntervalSeconds)             // recurring interval
        );
    }

    public async Task PollAsync(byte[] _)
    {
        var state = await StateManager.GetStateAsync<SubscriptionState>(StateKey);

        try
        {
            var status = await externalBillingApi.GetSubscriptionStatusAsync(state.SubscriptionId);
            state.LastChecked = DateTime.UtcNow;
            state.LastStatus = status;

            if (status == "past_due" && state.LastStatus != "past_due")
            {
                // Status changed - publish alert
                await daprClient.PublishEventAsync("pubsub", "subscription.past-due", new
                {
                    SubscriptionId = state.SubscriptionId,
                    DetectedAt = DateTime.UtcNow
                });
            }

            await StateManager.SetStateAsync(StateKey, state);
        }
        catch (Exception ex)
        {
            state.LastError = ex.Message;
            state.ErrorCount++;
            await StateManager.SetStateAsync(StateKey, state);
        }
    }

    public async Task StopMonitoringAsync()
    {
        await UnregisterTimerAsync(TimerName);
    }

    public async Task ReceiveReminderAsync(string _, ...) { }
}
```

## Managing Poll Actors from a Controller

```javascript
const { DaprClient } = require('@dapr/dapr');
const client = new DaprClient();

// Start monitoring a subscription every 60 seconds
async function startMonitoring(subscriptionId) {
  await client.actor.invoke(
    'SubscriptionMonitor',
    subscriptionId,
    'StartMonitoringAsync',
    { subscriptionId, pollIntervalSeconds: 60 }
  );
}

// Stop monitoring (e.g., on cancellation)
async function stopMonitoring(subscriptionId) {
  await client.actor.invoke(
    'SubscriptionMonitor',
    subscriptionId,
    'StopMonitoringAsync'
  );
}

// Start monitoring for all active subscriptions at startup
async function bootstrapMonitoring() {
  const subscriptions = await db.getActiveSubscriptions();
  await Promise.all(subscriptions.map(s => startMonitoring(s.id)));
  console.log(`Started monitoring ${subscriptions.length} subscriptions`);
}
```

## Adaptive Poll Interval

```csharp
public async Task PollAsync(byte[] _)
{
    var state = await StateManager.GetStateAsync<SubscriptionState>(StateKey);

    // Slow down polling if no changes detected for a while
    if ((DateTime.UtcNow - state.LastChangeDetected).TotalHours > 24)
    {
        await UnregisterTimerAsync(TimerName);
        await RegisterTimerAsync(TimerName, nameof(PollAsync), null,
            TimeSpan.Zero, TimeSpan.FromMinutes(15));  // poll every 15 min instead
    }

    // ... poll logic
}
```

## Summary

Dapr actor timers enable per-entity polling where each actor schedules its own check interval independently. This scales to millions of monitored entities without a centralized scheduler, and adaptive intervals reduce load for idle entities while keeping active ones responsive.
