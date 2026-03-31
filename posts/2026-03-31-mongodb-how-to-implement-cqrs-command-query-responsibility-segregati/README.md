# How to Implement CQRS with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, CQRS, Architecture, Node.js, Design Patterns

Description: Learn how to implement Command Query Responsibility Segregation (CQRS) with MongoDB using separate write and read models for optimized performance.

---

## What Is CQRS

CQRS (Command Query Responsibility Segregation) separates the model used to write data (commands) from the model used to read data (queries). In MongoDB this typically means:

- **Write model** - normalized, domain-accurate collections optimized for writes and consistency
- **Read model** - denormalized, query-optimized collections that aggregate data for specific views

This separation allows each side to be optimized independently.

## CQRS Architecture Overview

```text
Client
  |
  |-- Command (write intent) --> Command Handler
  |                                    |
  |                              Write Model (MongoDB)
  |                                    |
  |                             Domain Event Published
  |                                    |
  |                             Event Handler (Projector)
  |                                    |
  |                              Read Model (MongoDB)
  |                                    |
  |-- Query (read) ------------------> Query Handler
  |                                    |
  |                              Read Model (MongoDB)
```

## Write Model - Normalized Domain Collections

```javascript
// Write model: orders collection (normalized, write-optimized)
{
  _id: "order-123",
  customerId: "cust-456",
  status: "pending",
  lineItems: [
    { productId: "prod-789", qty: 2, unitPrice: 29.99 }
  ],
  createdAt: ISODate(),
  updatedAt: ISODate()
}

// customers collection (separate from orders)
{
  _id: "cust-456",
  name: "Alice",
  email: "alice@example.com",
  tier: "premium"
}
```

## Command Handlers

```javascript
// commands/placeOrder.js
class PlaceOrderCommand {
  constructor({ customerId, lineItems }) {
    this.customerId = customerId;
    this.lineItems = lineItems;
  }
}

class PlaceOrderHandler {
  constructor(db, eventBus) {
    this.orders = db.collection('orders');
    this.customers = db.collection('customers');
    this.eventBus = eventBus;
  }

  async handle(command) {
    // Validate customer exists
    const customer = await this.customers.findOne({ _id: command.customerId });
    if (!customer) throw new Error('Customer not found');

    // Calculate total
    const total = command.lineItems.reduce(
      (sum, item) => sum + item.qty * item.unitPrice, 0
    );

    // Write to write model
    const order = {
      _id: `order-${Date.now()}`,
      customerId: command.customerId,
      status: 'pending',
      lineItems: command.lineItems,
      total,
      createdAt: new Date(),
      updatedAt: new Date()
    };

    await this.orders.insertOne(order);

    // Publish domain event for read model sync
    await this.eventBus.publish('OrderPlaced', {
      orderId: order._id,
      customerId: order.customerId,
      customerName: customer.name,
      customerTier: customer.tier,
      lineItems: order.lineItems,
      total,
      placedAt: order.createdAt
    });

    return order._id;
  }
}
```

## Query Handlers

```javascript
// queries/getCustomerOrders.js
class GetCustomerOrdersQuery {
  constructor({ customerId, page = 1, limit = 20 }) {
    this.customerId = customerId;
    this.page = page;
    this.limit = limit;
  }
}

class GetCustomerOrdersHandler {
  constructor(db) {
    // Reads from the READ model - not the write model
    this.readModel = db.collection('order_summaries_read');
  }

  async handle(query) {
    const skip = (query.page - 1) * query.limit;

    const [items, total] = await Promise.all([
      this.readModel
        .find({ customerId: query.customerId })
        .sort({ placedAt: -1 })
        .skip(skip)
        .limit(query.limit)
        .toArray(),
      this.readModel.countDocuments({ customerId: query.customerId })
    ]);

    return {
      items,
      total,
      page: query.page,
      totalPages: Math.ceil(total / query.limit)
    };
  }
}
```

## Read Model - Denormalized for Queries

```javascript
// order_summaries_read collection (read-optimized, denormalized)
{
  _id: "order-123",
  customerId: "cust-456",
  customerName: "Alice",           // denormalized from customers
  customerTier: "premium",         // denormalized from customers
  status: "pending",
  productSummary: "Widget x2, Gadget x1",  // precomputed string
  lineItemCount: 2,
  total: 89.97,
  placedAt: ISODate(),
  lastUpdatedAt: ISODate()
}

// Index optimized for common query patterns
db.order_summaries_read.createIndex({ customerId: 1, placedAt: -1 });
db.order_summaries_read.createIndex({ status: 1, placedAt: -1 });
db.order_summaries_read.createIndex({ customerTier: 1, total: -1 });
```

## Event Bus and Projectors

```javascript
// projectors/orderProjector.js
class OrderProjector {
  constructor(db) {
    this.readModel = db.collection('order_summaries_read');
  }

  async onOrderPlaced(event) {
    const productSummary = event.lineItems
      .map(i => `${i.productName || i.productId} x${i.qty}`)
      .join(', ')
      .substring(0, 200);

    await this.readModel.insertOne({
      _id: event.orderId,
      customerId: event.customerId,
      customerName: event.customerName,
      customerTier: event.customerTier,
      status: 'pending',
      productSummary,
      lineItemCount: event.lineItems.length,
      total: event.total,
      placedAt: event.placedAt,
      lastUpdatedAt: new Date()
    });
  }

  async onOrderConfirmed(event) {
    await this.readModel.updateOne(
      { _id: event.orderId },
      { $set: { status: 'confirmed', confirmedAt: event.confirmedAt, lastUpdatedAt: new Date() } }
    );
  }

  async onOrderShipped(event) {
    await this.readModel.updateOne(
      { _id: event.orderId },
      {
        $set: {
          status: 'shipped',
          trackingNumber: event.trackingNumber,
          shippedAt: event.shippedAt,
          lastUpdatedAt: new Date()
        }
      }
    );
  }
}
```

## Command Bus - Routing Commands to Handlers

```javascript
// commandBus.js
class CommandBus {
  constructor() {
    this.handlers = new Map();
  }

  register(CommandClass, handler) {
    this.handlers.set(CommandClass.name, handler);
  }

  async dispatch(command) {
    const handlerName = command.constructor.name;
    const handler = this.handlers.get(handlerName);
    if (!handler) throw new Error(`No handler for command: ${handlerName}`);
    return handler.handle(command);
  }
}

// Wiring it together
const commandBus = new CommandBus();
commandBus.register(PlaceOrderCommand, new PlaceOrderHandler(db, eventBus));
commandBus.register(ConfirmOrderCommand, new ConfirmOrderHandler(db, eventBus));

// Usage in HTTP route
app.post('/api/orders', async (req, res) => {
  const commandId = await commandBus.dispatch(
    new PlaceOrderCommand(req.body)
  );
  res.status(202).json({ orderId: commandId });
});

app.get('/api/customers/:id/orders', async (req, res) => {
  const result = await queryHandler.handle(
    new GetCustomerOrdersQuery({ customerId: req.params.id, ...req.query })
  );
  res.json(result);
});
```

## Handling Eventual Consistency

Read models are eventually consistent - they update asynchronously after commands complete:

```javascript
// For commands that need to confirm the read model is updated
async function waitForReadModel(readModel, orderId, maxWaitMs = 5000) {
  const start = Date.now();
  while (Date.now() - start < maxWaitMs) {
    const doc = await readModel.findOne({ _id: orderId });
    if (doc) return doc;
    await new Promise(resolve => setTimeout(resolve, 100));
  }
  throw new Error('Read model did not update in time');
}
```

## Summary

CQRS with MongoDB separates commands (writes to normalized domain collections) from queries (reads from denormalized read model collections). Commands emit domain events that projectors consume to build and maintain the read model asynchronously. This allows you to optimize each side independently - write models focus on domain integrity while read models are denormalized and indexed for specific query access patterns. The key tradeoff is eventual consistency between write and read models, which is acceptable for most query use cases.
