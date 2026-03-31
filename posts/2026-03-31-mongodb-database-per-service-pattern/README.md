# How to Implement Database per Service with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Microservice, Database, Pattern, Isolation

Description: Learn how to implement the database-per-service pattern with MongoDB, including namespace strategies, access control, and avoiding shared-database pitfalls.

---

The database-per-service pattern is a cornerstone of microservices design. Each service owns its data exclusively - no other service can access its database directly. MongoDB supports this pattern through multiple isolation strategies, from separate Atlas clusters to namespace-level separation within a shared cluster.

## Why Database Isolation Matters

When multiple services share the same database, a schema change in one service can break another. Deployments become coupled and independent scaling is harder. Database-per-service removes these constraints: each service team can evolve their data model without coordinating with others.

## Isolation Strategy Options

There are three levels of isolation with MongoDB:

1. **Separate clusters** - Maximum isolation, highest cost
2. **Separate databases on a shared cluster** - Good balance for most teams
3. **Separate collections with naming conventions** - Lowest cost, requires discipline

For most production systems, separate databases on a shared Atlas cluster is the right choice.

## Setting Up Separate Databases

Each service gets its own database. Enforce this by creating a dedicated MongoDB user per service with access restricted to only that service's database:

```javascript
// Run in mongosh as admin
db.createUser({
  user: "order_service",
  pwd: "strongpassword",
  roles: [
    { role: "readWrite", db: "order_service_db" }
  ]
});

db.createUser({
  user: "inventory_service",
  pwd: "strongpassword2",
  roles: [
    { role: "readWrite", db: "inventory_service_db" }
  ]
});
```

Now even if a developer makes a mistake in the order service code, it cannot accidentally read or write inventory data.

## Configuring Service Connection Strings

Each service uses its own dedicated connection string with scoped credentials:

```bash
# Order service .env
MONGO_URI=mongodb+srv://order_service:strongpassword@cluster.mongodb.net/order_service_db

# Inventory service .env
MONGO_URI=mongodb+srv://inventory_service:strongpassword2@cluster.mongodb.net/inventory_service_db
```

## Schema Evolution Without Coordination

One of the main benefits of database-per-service is independent schema evolution. Add fields using nullable patterns so old and new service versions coexist during rolling deployments:

```javascript
// New version of order service adds 'region' field
// Old documents simply won't have this field - handle with defaults
async function getOrder(db, orderId) {
  const order = await db.collection('orders').findOne({ _id: orderId });
  return {
    ...order,
    region: order.region ?? 'us-east-1'  // default for legacy documents
  };
}
```

## Handling Data That Belongs to Multiple Services

When a query seems to need data from multiple service databases, resist the urge to join directly. Instead, use service-to-service API calls or maintain a read model:

```python
# Inventory service exposes its own API
# Order service calls it when needed - never accesses inventory DB directly
async def reserve_inventory(product_id: str, quantity: int):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://inventory-service/reserve",
            json={"productId": product_id, "quantity": quantity}
        )
        response.raise_for_status()
        return response.json()
```

## Monitoring Per-Service Database Usage

Track storage and operation metrics per database to identify which services are growing fastest:

```javascript
// In mongosh - check database stats per service
db.adminCommand({ listDatabases: 1 }).databases.forEach(d => {
  const stats = db.getSiblingDB(d.name).stats();
  print(`${d.name}: ${(stats.dataSize / 1024 / 1024).toFixed(2)} MB`);
});
```

## Summary

Implementing database-per-service with MongoDB gives each microservice true data autonomy. By creating separate databases with scoped credentials per service, teams can evolve schemas independently, scale databases individually, and deploy services without cross-team coordination. The key discipline is enforcing that no service accesses another service's database - all cross-service data access must go through APIs.
