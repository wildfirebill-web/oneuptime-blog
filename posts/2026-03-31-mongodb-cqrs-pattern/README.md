# How to Implement CQRS with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, CQRS, Architecture, Pattern, Change Stream

Description: Learn how to implement CQRS (Command Query Responsibility Segregation) with MongoDB by separating write and read models using change streams.

---

## What Is CQRS?

CQRS separates the write model (commands that mutate state) from the read model (queries that return data). The two models can have different schemas, different databases, and different scaling strategies. In MongoDB, the write model stores normalized event-like documents while the read model maintains denormalized, query-optimized projections.

## Why CQRS with MongoDB?

MongoDB's change streams make it natural to project write-model changes to read models in real time. Write operations go to one collection, and a change stream listener updates one or more read-optimized collections automatically.

## The Write Model

The write model stores commands as granular, normalized documents:

```javascript
// orders collection (write model)
{
  "_id": "order-123",
  "customerId": "cust-456",
  "status": "pending",
  "items": [
    { "sku": "WIDGET-A", "qty": 2, "unitPrice": 9.99 },
    { "sku": "GADGET-X", "qty": 1, "unitPrice": 49.99 }
  ],
  "createdAt": ISODate("2026-03-31T10:00:00Z"),
  "updatedAt": ISODate("2026-03-31T10:00:00Z")
}
```

## The Read Model

The read model is denormalized for fast queries - it includes computed fields and denormalized customer data:

```javascript
// order_views collection (read model)
{
  "_id": "order-123",
  "customerName": "Alice Johnson",
  "customerEmail": "alice@example.com",
  "totalAmount": 69.97,
  "itemCount": 3,
  "status": "pending",
  "createdAt": ISODate("2026-03-31T10:00:00Z")
}
```

## Projecting Changes with Change Streams

```python
from pymongo import MongoClient
from threading import Thread

client = MongoClient("mongodb://localhost:27017/")
write_db = client.write_db
read_db = client.read_db

def project_order(doc: dict) -> dict:
    total = sum(
        item["qty"] * item["unitPrice"]
        for item in doc.get("items", [])
    )
    item_count = sum(item["qty"] for item in doc.get("items", []))
    return {
        "_id": doc["_id"],
        "totalAmount": round(total, 2),
        "itemCount": item_count,
        "status": doc["status"],
        "customerId": doc["customerId"],
        "createdAt": doc.get("createdAt")
    }

def listen_for_changes():
    with write_db.orders.watch(full_document="updateLookup") as stream:
        for change in stream:
            op = change["operationType"]
            if op in ("insert", "update", "replace"):
                doc = change["fullDocument"]
                projection = project_order(doc)
                read_db.order_views.replace_one(
                    {"_id": doc["_id"]},
                    projection,
                    upsert=True
                )
            elif op == "delete":
                doc_id = change["documentKey"]["_id"]
                read_db.order_views.delete_one({"_id": doc_id})

Thread(target=listen_for_changes, daemon=True).start()
```

## Command Handler

```python
from datetime import datetime, timezone

def create_order(customer_id: str, items: list) -> str:
    from bson import ObjectId
    order_id = str(ObjectId())
    now = datetime.now(timezone.utc)

    write_db.orders.insert_one({
        "_id": order_id,
        "customerId": customer_id,
        "status": "pending",
        "items": items,
        "createdAt": now,
        "updatedAt": now
    })
    return order_id

def update_order_status(order_id: str, new_status: str) -> None:
    write_db.orders.update_one(
        {"_id": order_id},
        {
            "$set": {
                "status": new_status,
                "updatedAt": datetime.now(timezone.utc)
            }
        }
    )
```

## Query Handler

```python
def get_order_summary(order_id: str) -> dict | None:
    return read_db.order_views.find_one({"_id": order_id})

def list_pending_orders(limit: int = 20) -> list:
    return list(read_db.order_views.find(
        {"status": "pending"},
        sort=[("createdAt", -1)],
        limit=limit
    ))
```

## Scaling Read and Write Models Independently

With CQRS, you can:
- Add read replicas only for the read model database to scale query throughput
- Use different indexes on write and read collections
- Deploy the projection listener as a separate service that scales independently

## Summary

CQRS in MongoDB uses two separate collections: a write model for command operations and a read model for queries. A change stream listener projects write-model changes to the read model in real time. Commands mutate the write model, queries read from the optimized read model, and the two models can evolve and scale independently.
