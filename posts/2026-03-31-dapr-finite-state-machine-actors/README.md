# How to Implement Finite State Machine with Dapr Actors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Finite State Machine, Workflow, State Management

Description: Learn how to implement a finite state machine using Dapr actors to model entity lifecycle with explicit states, allowed transitions, and transition side effects.

---

## Why Use Actors for FSM?

A finite state machine (FSM) tracks an entity through a well-defined sequence of states. Dapr actors enforce turn-based concurrency, so no two transitions can run concurrently on the same entity. Combined with durable state, the current state survives crashes and restarts.

## Order FSM States and Transitions

```text
draft -> confirmed -> paid -> shipped -> delivered
                 |
                 v
             cancelled
```

## FSM Actor Interface

```typescript
interface IOrderFSMActor {
  transition(event: string, payload?: any): Promise<{ success: boolean; currentState: string }>;
  getState(): Promise<string>;
  getHistory(): Promise<TransitionRecord[]>;
}
```

## FSM Actor Implementation

```javascript
// Using Dapr JS SDK actor pattern
class OrderFSMActor {
  constructor(host) {
    this.host = host;
    this.stateManager = host.stateManager;
  }

  // Valid transitions: { fromState: { event: toState } }
  TRANSITIONS = {
    draft: { confirm: 'confirmed', cancel: 'cancelled' },
    confirmed: { pay: 'paid', cancel: 'cancelled' },
    paid: { ship: 'shipped' },
    shipped: { deliver: 'delivered' }
  };

  async transition(event, payload = {}) {
    const currentState = await this.stateManager.get('state') || 'draft';
    const allowed = this.TRANSITIONS[currentState];

    if (!allowed || !allowed[event]) {
      return {
        success: false,
        currentState,
        error: `Transition '${event}' not allowed from state '${currentState}'`
      };
    }

    const nextState = allowed[event];

    // Execute side effect for this transition
    await this.onTransition(currentState, event, nextState, payload);

    // Record transition history
    const history = await this.stateManager.get('history') || [];
    history.push({
      from: currentState,
      event,
      to: nextState,
      at: new Date().toISOString(),
      payload
    });

    // Save new state and history atomically
    await this.stateManager.set('state', nextState);
    await this.stateManager.set('history', history);

    return { success: true, currentState: nextState };
  }

  async onTransition(from, event, to, payload) {
    if (to === 'confirmed') {
      console.log(`Order ${this.host.id} confirmed - reserving inventory`);
      // Trigger downstream actions
    }
    if (to === 'cancelled') {
      console.log(`Order ${this.host.id} cancelled - reason: ${payload.reason}`);
    }
  }

  async getState() {
    return await this.stateManager.get('state') || 'draft';
  }

  async getHistory() {
    return await this.stateManager.get('history') || [];
  }
}
```

## Triggering Transitions via HTTP

```javascript
const { DaprClient } = require('@dapr/dapr');
const client = new DaprClient();

app.post('/orders/:orderId/confirm', async (req, res) => {
  const result = await client.actor.invoke(
    'OrderFSMActor', req.params.orderId,
    'transition', { event: 'confirm' }
  );
  res.json(result);
});

app.post('/orders/:orderId/pay', async (req, res) => {
  const result = await client.actor.invoke(
    'OrderFSMActor', req.params.orderId,
    'transition', { event: 'pay', payload: { paymentId: req.body.paymentId } }
  );
  res.json(result);
});
```

## Querying FSM State and History

```bash
# Get current state
curl http://localhost:3500/v1.0/actors/OrderFSMActor/order-456/method/getState

# Get full transition history
curl http://localhost:3500/v1.0/actors/OrderFSMActor/order-456/method/getHistory
```

## Summary

Dapr actors implement FSMs with durable state, turn-based concurrency, and a clean transition table. Invalid transitions are rejected immediately, history is appended on each successful transition, and the actor lifecycle ensures only one transition executes at a time per entity - eliminating concurrent state mutation bugs.
