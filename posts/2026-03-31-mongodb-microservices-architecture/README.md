# How to Use MongoDB in a Microservices Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Microservice, Architecture, Database, Scalability

Description: Learn how to effectively use MongoDB in a microservices architecture, covering service isolation, data modeling, and connection management.

---

MongoDB's flexible document model makes it a natural fit for microservices architectures where each service owns its data and schema can evolve independently. This guide covers the key patterns and practices for integrating MongoDB into your microservices stack.

## Why MongoDB Works Well with Microservices

In a microservices system, each service should be loosely coupled and independently deployable. MongoDB supports this through its schema-less design, horizontal scaling via sharding, and built-in replication. Each microservice can maintain its own MongoDB database or collection namespace without tight coupling to other services.

Key advantages:
- No rigid schema migrations required when individual services evolve
- Each service can use its own database on a shared cluster
- Rich query capabilities reduce the need for cross-service data fetching
- Change streams enable event-driven communication between services

## Setting Up Service-Specific Databases

The recommended pattern is one database per service. Here is how to configure separate MongoDB connections in a Node.js microservice:

```javascript
const { MongoClient } = require('mongodb');

const ORDER_SERVICE_URI = process.env.MONGO_URI;
const DB_NAME = 'order_service';

let db;

async function connect() {
  const client = new MongoClient(ORDER_SERVICE_URI, {
    maxPoolSize: 10,
    minPoolSize: 2,
    connectTimeoutMS: 5000,
  });
  await client.connect();
  db = client.db(DB_NAME);
  return db;
}

module.exports = { connect, getDb: () => db };
```

Each service defines its own URI and database name via environment variables, keeping configurations isolated.

## Data Modeling for Microservices

Avoid embedding cross-service references as full objects. Instead, store only the foreign ID and fetch additional data through service APIs when needed. This keeps data ownership clear.

```javascript
// Order service document - stores only user ID, not full user object
const orderDoc = {
  _id: new ObjectId(),
  userId: "usr_abc123",       // reference to user service
  productIds: ["prod_001"],   // reference to product service
  total: 99.99,
  status: "pending",
  createdAt: new Date()
};
```

## Handling Cross-Service Reads

When a service needs data from another service, use API calls rather than direct database access. This preserves service boundaries and allows each service to evolve its schema independently.

```python
import httpx

async def get_order_with_user_details(order_id: str):
    async with httpx.AsyncClient() as client:
        order_resp = await client.get(f"http://order-service/orders/{order_id}")
        order = order_resp.json()

        user_resp = await client.get(f"http://user-service/users/{order['userId']}")
        user = user_resp.json()

    return {**order, "user": user}
```

## Connection Pooling Best Practices

Microservices often run many replicas. Misconfigured connection pools can exhaust MongoDB's connection limit. Configure pools conservatively:

```yaml
# docker-compose environment variables
environment:
  MONGO_MAX_POOL_SIZE: "10"
  MONGO_MIN_POOL_SIZE: "2"
  MONGO_MAX_IDLE_TIME_MS: "30000"
```

With 50 service replicas and a pool size of 10, you could open 500 connections. Monitor with `db.serverStatus().connections` and keep the total under your Atlas or server limit.

## Using Change Streams for Event-Driven Communication

MongoDB change streams let one service react to data changes in another service's database (when sharing a cluster) or in its own database to publish domain events:

```javascript
async function watchOrders(db) {
  const orders = db.collection('orders');
  const changeStream = orders.watch([
    { $match: { 'operationType': 'insert', 'fullDocument.status': 'pending' } }
  ]);

  changeStream.on('change', async (event) => {
    const order = event.fullDocument;
    // Publish to message broker for other services to consume
    await publishEvent('order.created', order);
  });
}
```

## Summary

MongoDB integrates well into microservices architectures when each service owns its own database, uses connection pools responsibly, and communicates through APIs or message brokers rather than direct cross-database access. Change streams provide a powerful mechanism for event-driven patterns, while the flexible schema allows independent service evolution without coordinated migrations.
