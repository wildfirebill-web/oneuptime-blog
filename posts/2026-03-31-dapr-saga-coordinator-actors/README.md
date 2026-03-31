# How to Implement Saga Coordinator with Dapr Actors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Saga, Orchestration, Distributed Transaction

Description: Learn how to implement a saga coordinator using Dapr actors to orchestrate multi-step distributed transactions and drive compensations when steps fail.

---

## Orchestration vs Choreography Sagas

In a choreography saga, services react to events with no central controller. In an orchestration saga, a coordinator drives the workflow. The coordinator pattern is simpler to debug since all saga logic is in one place. Dapr actors are ideal coordinators: stateful, durable, and invocable from anywhere.

## Saga State

```javascript
const SAGA_STATES = {
  STARTED: 'started',
  INVENTORY_RESERVED: 'inventory_reserved',
  PAYMENT_CHARGED: 'payment_charged',
  SHIPMENT_CREATED: 'shipment_created',
  COMPLETED: 'completed',
  COMPENSATING: 'compensating',
  FAILED: 'failed'
};
```

## Order Saga Coordinator Actor

```javascript
class OrderSagaActor {
  constructor(host) {
    this.stateManager = host.stateManager;
    this.actorId = host.id;
    this.client = new (require('@dapr/dapr').DaprClient)();
  }

  async start(order) {
    await this.stateManager.set('saga', {
      orderId: order.id,
      order,
      state: SAGA_STATES.STARTED,
      history: [],
      startedAt: Date.now()
    });

    await this.executeStep();
  }

  async executeStep() {
    const saga = await this.stateManager.get('saga');

    switch (saga.state) {
      case SAGA_STATES.STARTED:
        await this.reserveInventory(saga);
        break;
      case SAGA_STATES.INVENTORY_RESERVED:
        await this.chargePayment(saga);
        break;
      case SAGA_STATES.PAYMENT_CHARGED:
        await this.createShipment(saga);
        break;
      case SAGA_STATES.SHIPMENT_CREATED:
        await this.completeOrder(saga);
        break;
    }
  }

  async reserveInventory(saga) {
    try {
      const reservation = await this.client.invokeMethod(
        'inventory-service', 'reserve', 'POST', { items: saga.order.items }
      );
      await this.updateSagaState(SAGA_STATES.INVENTORY_RESERVED, { reservationId: reservation.id });
      await this.executeStep();
    } catch (err) {
      await this.startCompensation(saga, 'inventory_failed', err.message);
    }
  }

  async chargePayment(saga) {
    try {
      const charge = await this.client.invokeMethod(
        'payment-service', 'charge', 'POST',
        { orderId: saga.orderId, amount: saga.order.totalAmount }
      );
      await this.updateSagaState(SAGA_STATES.PAYMENT_CHARGED, { chargeId: charge.id });
      await this.executeStep();
    } catch (err) {
      await this.startCompensation(saga, 'payment_failed', err.message);
    }
  }

  async startCompensation(saga, reason, error) {
    await this.updateSagaState(SAGA_STATES.COMPENSATING, { reason, error });

    // Compensate in reverse order
    if ([SAGA_STATES.PAYMENT_CHARGED, SAGA_STATES.SHIPMENT_CREATED].includes(saga.state)) {
      await this.client.invokeMethod('payment-service', 'refund', 'POST', {
        chargeId: saga.context.chargeId
      });
    }
    if ([SAGA_STATES.INVENTORY_RESERVED, SAGA_STATES.PAYMENT_CHARGED].includes(saga.state)) {
      await this.client.invokeMethod('inventory-service', 'release', 'POST', {
        reservationId: saga.context.reservationId
      });
    }

    await this.updateSagaState(SAGA_STATES.FAILED);
  }

  async updateSagaState(newState, context = {}) {
    const saga = await this.stateManager.get('saga');
    saga.history.push({ state: saga.state, at: Date.now() });
    saga.state = newState;
    saga.context = { ...(saga.context || {}), ...context };
    await this.stateManager.set('saga', saga);
  }
}
```

## Starting a Saga

```javascript
const { DaprClient } = require('@dapr/dapr');
const client = new DaprClient();

app.post('/orders', async (req, res) => {
  const order = await createOrderRecord(req.body);

  // Each order gets its own coordinator actor
  await client.actor.invoke('OrderSagaActor', order.id, 'start', order);

  res.json({ orderId: order.id, status: 'processing' });
});
```

## Checking Saga Status

```javascript
app.get('/orders/:orderId/saga-status', async (req, res) => {
  const saga = await client.actor.invoke(
    'OrderSagaActor', req.params.orderId, 'getSagaState'
  );
  res.json(saga);
});
```

## Summary

A Dapr actor saga coordinator centralizes all orchestration logic in one stateful entity per transaction. Step sequencing, error detection, and compensation are all driven by the coordinator, making it easy to inspect the current saga state, replay history, and add new steps without changing participant services.
