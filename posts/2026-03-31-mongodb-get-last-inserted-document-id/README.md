# How to Get the Last Inserted Document ID in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Insert, ObjectId, Driver

Description: Learn how to capture the _id of the last inserted document in MongoDB using insertOne, insertMany, and various language drivers.

---

## How MongoDB Assigns Document IDs

Every document inserted into MongoDB receives an `_id` field. If you do not provide one, the driver generates an `ObjectId` automatically before sending the insert to the server. Because the ID is created client-side, the driver already knows the value the moment the insert returns - you do not need a secondary query to retrieve it.

## Getting the ID After insertOne

### mongosh

```javascript
const result = db.orders.insertOne({ item: "widget", qty: 10 });
print(result.insertedId);
```

`insertOne` returns an object with a single `insertedId` property.

### Node.js

```javascript
const { MongoClient, ObjectId } = require("mongodb");

const client = new MongoClient("mongodb://localhost:27017");
await client.connect();
const db = client.db("shop");

const result = await db.collection("orders").insertOne({
  item: "widget",
  qty: 10,
  createdAt: new Date(),
});

console.log("Inserted ID:", result.insertedId.toString());
// Output: Inserted ID: 64abc1234567890abcdef012
```

### Python (PyMongo)

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
db = client["shop"]

result = db.orders.insert_one({"item": "widget", "qty": 10})
print("Inserted ID:", result.inserted_id)
```

### Go

```go
result, err := collection.InsertOne(ctx, bson.D{
    {Key: "item", Value: "widget"},
    {Key: "qty", Value: 10},
})
if err != nil {
    log.Fatal(err)
}
insertedID := result.InsertedID.(primitive.ObjectID)
fmt.Println("Inserted ID:", insertedID.Hex())
```

## Getting IDs After insertMany

When inserting multiple documents at once, the driver returns an `insertedIds` map (index to ID):

```javascript
const result = db.products.insertMany([
  { name: "Alpha" },
  { name: "Beta" },
  { name: "Gamma" },
]);

// insertedIds is a map: { 0: ObjectId(...), 1: ObjectId(...), 2: ObjectId(...) }
Object.values(result.insertedIds).forEach((id) => print(id));
```

In Python, the equivalent property is `inserted_ids`, a list:

```python
result = db.products.insert_many([
    {"name": "Alpha"},
    {"name": "Beta"},
    {"name": "Gamma"},
])

for oid in result.inserted_ids:
    print(oid)
```

## Extracting Timestamp from ObjectId

An `ObjectId` embeds the insertion timestamp in its first four bytes, so you can derive "when was this document inserted" without a separate field:

```javascript
const id = result.insertedId;
const insertionTime = id.getTimestamp();
// Returns a Date object
print(insertionTime.toISOString());
```

In Python:

```python
from bson import ObjectId

oid = result.inserted_id
print(oid.generation_time)  # datetime in UTC
```

## Using a Custom ID

If you supply your own `_id`, the returned `insertedId` simply reflects what you provided:

```javascript
const result = db.users.insertOne({ _id: "user-42", name: "Alice" });
print(result.insertedId); // "user-42"
```

## Summary

MongoDB drivers expose the inserted ID directly on the result object returned by `insertOne` (`result.insertedId`) and `insertMany` (`result.insertedIds`). Because IDs are generated client-side before the network round-trip, no extra query is needed. For bulk inserts, iterate over `insertedIds` to get each document's new ID. If you need the creation timestamp, parse it directly from the `ObjectId` using `getTimestamp()` or `generation_time`.
