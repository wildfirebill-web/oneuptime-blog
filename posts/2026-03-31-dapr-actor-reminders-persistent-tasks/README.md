# How to Use Dapr Actor Reminders for Persistent Scheduled Tasks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Reminder, Scheduling, Persistence

Description: Learn how to use Dapr actor reminders to create persistent scheduled tasks that survive actor deactivation and application restarts, with configuration examples.

---

## Introduction

Dapr actor reminders are persistent, durable scheduled callbacks associated with an actor. Unlike timers, reminders are stored in the actor's state store and survive actor deactivation, host failures, and application restarts. When an actor with a pending reminder is reactivated, the reminder fires as soon as the actor comes back online.

Reminders are ideal for:

- Subscription renewal notifications
- Scheduled billing or invoicing
- Deferred processing of business events
- Any task that must happen even if the actor was inactive

## How Reminders Differ from Timers

```mermaid
flowchart TB
    subgraph Timer
        T1[Actor Active] --> T2[Register Timer]
        T2 --> T3[Timer Fires Periodically]
        T3 --> T4[Actor Deactivated]
        T4 --> T5[Timer Cancelled]
    end

    subgraph Reminder
        R1[Actor Active] --> R2[Register Reminder]
        R2 --> R3[Reminder Stored in State Store]
        R3 --> R4[Actor Deactivated]
        R4 --> R5[Reminder Persisted]
        R5 --> R6[Actor Reactivated]
        R6 --> R7[Reminder Fires Again]
    end
```

## Prerequisites

- Dapr initialized locally or on Kubernetes
- State store component with `actorStateStore: "true"`
- An actor application running with Dapr sidecar

## Registering a Reminder

### Via HTTP API

```bash
curl -X POST \
  http://localhost:3500/v1.0/actors/SubscriptionActor/sub-001/reminders/renewalAlert \
  -H "Content-Type: application/json" \
  -d '{
    "dueTime": "72h",
    "period": "24h",
    "data": {"planId": "pro", "userId": "user-42"},
    "ttl": "168h"
  }'
```

Parameters:
- `dueTime` - delay before the first fire (supports ISO 8601 and Go duration strings)
- `period` - interval between subsequent fires (omit for a one-shot reminder)
- `data` - arbitrary JSON payload passed to the callback
- `ttl` - optional time-to-live after which the reminder expires

### Via Go SDK

```go
package main

import (
    "context"
    "encoding/json"
    "github.com/dapr/go-sdk/actor"
)

type SubscriptionActorImpl struct {
    actor.ServerImplBase
}

func (a *SubscriptionActorImpl) Type() string { return "SubscriptionActor" }

// Register reminder on activation
func (a *SubscriptionActorImpl) OnActivate() error {
    data, _ := json.Marshal(map[string]string{
        "planId": "pro",
        "userId": "user-42",
    })
    return a.RegisterActorReminder("renewalAlert", data, "72h", "24h")
}

// Reminder callback - Dapr calls the method matching the reminder name
func (a *SubscriptionActorImpl) RenewalAlert(ctx context.Context, data []byte) error {
    var payload map[string]string
    json.Unmarshal(data, &payload)
    planId := payload["planId"]
    userId := payload["userId"]
    // Send renewal notification logic
    _ = planId
    _ = userId
    return nil
}
```

### Via Python SDK

```python
from dapr.actor import Actor, ActorInterface, actormethod

class SubscriptionActorInterface(ActorInterface):
    @actormethod(name="renewalAlert")
    async def renewal_alert(self, data: dict) -> None: ...

class SubscriptionActor(Actor, SubscriptionActorInterface):
    async def _on_activate(self) -> None:
        await self.register_reminder(
            reminder_name="renewalAlert",
            state={"planId": "pro", "userId": "user-42"},
            due_time="72h",
            period="24h"
        )

    async def renewal_alert(self, data: dict) -> None:
        plan_id = data.get("planId")
        user_id = data.get("userId")
        print(f"Sending renewal notice to user {user_id} for plan {plan_id}")
```

## Handling Reminder Callbacks (HTTP, No SDK)

When using a raw HTTP server, Dapr calls the reminder as a PUT to your actor method endpoint:

```javascript
// Node.js Express
app.put('/actors/SubscriptionActor/:actorId/method/renewalAlert', (req, res) => {
  const { actorId } = req.params;
  const reminderBody = req.body; // Contains data, dueTime, period
  const { data } = reminderBody;
  console.log(`Reminder fired for actor ${actorId}`, data);
  res.sendStatus(200);
});
```

## Getting a Reminder

```bash
curl http://localhost:3500/v1.0/actors/SubscriptionActor/sub-001/reminders/renewalAlert
```

Response:

```json
{
  "period": "24h",
  "dueTime": "2026-04-03T10:00:00Z",
  "data": {"planId": "pro", "userId": "user-42"}
}
```

## Deleting a Reminder

```bash
curl -X DELETE \
  http://localhost:3500/v1.0/actors/SubscriptionActor/sub-001/reminders/renewalAlert
```

Reminders can also be deleted from within the actor code using the SDK's `UnregisterReminder` or `delete_reminder` methods.

## One-Shot Reminders

To create a reminder that fires only once, omit the `period` field:

```bash
curl -X POST \
  http://localhost:3500/v1.0/actors/OrderActor/order-9900/reminders/shipmentDue \
  -H "Content-Type: application/json" \
  -d '{
    "dueTime": "48h",
    "data": {"orderId": "order-9900", "courier": "FedEx"}
  }'
```

## Reminder Persistence Architecture

```mermaid
flowchart LR
    A[Actor App] -->|Register reminder| B[Dapr Sidecar]
    B -->|Save to state store| C[(State Store)]
    B -->|Register with| D[Placement Service]
    D -->|Actor host lookup| B
    B -->|Fire callback| A
    C -->|Reload on restart| B
```

## Summary

Dapr actor reminders are durable, persistent scheduled callbacks that survive actor deactivation and application restarts. By storing reminder state in the backing state store, Dapr guarantees that scheduled tasks are not lost. Use reminders for critical business workflows - such as subscription renewals, deferred processing, and deadline management - where missed execution is not acceptable. For transient, non-critical scheduling, prefer the lighter-weight timer mechanism.
