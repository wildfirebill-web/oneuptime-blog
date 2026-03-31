# How to Implement the Saga Pattern as an Alternative to Transactions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Saga Pattern, Distributed System, Microservice, Transaction

Description: Learn how to implement the Saga pattern in MongoDB as an alternative to multi-document transactions, using choreography and orchestration approaches.

---

Multi-document transactions in MongoDB require a replica set and add latency. For distributed microservices or write-heavy workloads, the Saga pattern offers a scalable alternative by breaking a logical transaction into a sequence of local operations with compensating actions on failure.

## When to Use Sagas Instead of Transactions

- Operations span multiple services or databases
- You want higher throughput without distributed lock overhead
- Long-running workflows where transaction timeouts are a concern
- Event-driven architectures using message queues

## Saga Approaches

**Choreography** - each service publishes events that trigger the next step.
**Orchestration** - a central coordinator drives the workflow.

This guide uses the orchestration approach with a dedicated `sagas` collection in MongoDB.

## Data Model

```javascript
// sagas collection
{
  _id: ObjectId(),
  type: "place_order",
  status: "pending", // pending | completed | compensating | failed
  steps: [
    { name: "reserve_inventory", status: "pending" },
    { name: "charge_payment",    status: "pending" },
    { name: "create_order",      status: "pending" }
  ],
  payload: { userId: "u1", items: [...] },
  createdAt: ISODate()
}
```

## Creating a Saga

```javascript
async function startOrderSaga(db, userId, items) {
  const saga = await db.collection('sagas').insertOne({
    type: 'place_order',
    status: 'pending',
    steps: [
      { name: 'reserve_inventory', status: 'pending' },
      { name: 'charge_payment',    status: 'pending' },
      { name: 'create_order',      status: 'pending' }
    ],
    payload: { userId, items },
    createdAt: new Date()
  });
  return saga.insertedId;
}
```

## Executing Steps

```javascript
async function executeStep(db, sagaId, stepName, fn) {
  // Mark step as running
  await db.collection('sagas').updateOne(
    { _id: sagaId, 'steps.name': stepName },
    { $set: { 'steps.$.status': 'running' } }
  );

  try {
    await fn();
    await db.collection('sagas').updateOne(
      { _id: sagaId, 'steps.name': stepName },
      { $set: { 'steps.$.status': 'completed' } }
    );
  } catch (err) {
    await db.collection('sagas').updateOne(
      { _id: sagaId, 'steps.name': stepName },
      { $set: { 'steps.$.status': 'failed', 'steps.$.error': err.message } }
    );
    await triggerCompensation(db, sagaId);
    throw err;
  }
}
```

## Compensating Transactions

Each step must have a compensating action that undoes its effect:

```javascript
const compensations = {
  reserve_inventory: async (db, payload) => {
    await db.collection('inventory').updateMany(
      { productId: { $in: payload.items.map(i => i.productId) } },
      { $inc: { quantity: /* restore amounts */ 1 } }
    );
  },
  charge_payment: async (db, payload) => {
    await db.collection('payments').insertOne({
      type: 'refund',
      userId: payload.userId,
      reason: 'saga_rollback'
    });
  }
};

async function triggerCompensation(db, sagaId) {
  const saga = await db.collection('sagas').findOne({ _id: sagaId });
  const completed = saga.steps.filter(s => s.status === 'completed').reverse();

  for (const step of completed) {
    const compensate = compensations[step.name];
    if (compensate) await compensate(db, saga.payload);
  }

  await db.collection('sagas').updateOne(
    { _id: sagaId },
    { $set: { status: 'failed' } }
  );
}
```

## Idempotency is Critical

Each step must be idempotent to survive retries. Use a unique identifier per step execution:

```javascript
await db.collection('inventory').updateOne(
  {
    productId: item.productId,
    processedSagas: { $ne: sagaId.toString() }
  },
  {
    $inc: { quantity: -item.qty },
    $push: { processedSagas: sagaId.toString() }
  }
);
```

## Recovery for Incomplete Sagas

Run a background job to recover stalled sagas:

```bash
# MongoDB shell - find sagas stuck in pending for over 5 minutes
db.sagas.find({
  status: "pending",
  createdAt: { $lt: new Date(Date.now() - 5 * 60 * 1000) }
})
```

## Summary

The Saga pattern replaces distributed transactions with local operations and compensating actions tracked in a MongoDB collection. Each step must be idempotent to support retries, and every completed step needs a corresponding compensation function for rollback. A background recovery process ensures sagas stuck in intermediate states are detected and resolved.

