# How to Use MongoDB Java Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Java, Driver, Backend Development, Programming

Description: Learn how to use the official MongoDB Java driver to connect, perform CRUD operations, run aggregations, manage transactions, and follow production best practices.

---

## Overview

The official MongoDB Java driver is the primary library for connecting Java applications to MongoDB. It supports both synchronous and reactive (async) APIs. The synchronous driver is covered here as it is the most commonly used.

```mermaid
flowchart LR
    App[Java Application] --> MC[MongoClient]
    MC -->|Connection Pool| MongoDB
    MC --> DB[getDatabase]
    DB --> COL[getCollection]
    COL --> OPS[CRUD + Aggregation + Transactions]
```

## Maven Dependency

Add to your `pom.xml`:

```xml
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver-sync</artifactId>
    <version>5.1.0</version>
</dependency>
```

For Gradle, add to `build.gradle`:

```text
implementation 'org.mongodb:mongodb-driver-sync:5.1.0'
```

## Connecting to MongoDB

Create a single `MongoClient` instance and share it across the application (it is thread-safe and manages a connection pool internally).

```java
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.MongoClientSettings;
import com.mongodb.ConnectionString;

// Simple connection
MongoClient client = MongoClients.create("mongodb://admin:password@127.0.0.1:27017/?authSource=admin");

// With explicit settings
MongoClientSettings settings = MongoClientSettings.builder()
    .applyConnectionString(new ConnectionString("mongodb://admin:password@127.0.0.1:27017/?authSource=admin"))
    .applyToConnectionPoolSettings(builder ->
        builder.maxSize(10).minSize(2)
    )
    .applyToSocketSettings(builder ->
        builder.connectTimeout(10, java.util.concurrent.TimeUnit.SECONDS)
               .readTimeout(45, java.util.concurrent.TimeUnit.SECONDS)
    )
    .build();

MongoClient client = MongoClients.create(settings);
```

## Getting a Database and Collection

```java
import com.mongodb.client.MongoDatabase;
import com.mongodb.client.MongoCollection;
import org.bson.Document;

MongoDatabase db = client.getDatabase("myapp");
MongoCollection<Document> orders = db.getCollection("orders");
```

## Insert Operations

```java
import org.bson.Document;
import java.util.Arrays;
import java.util.Date;

// Insert one
Document order = new Document("customerId", "c123")
    .append("total", 99.98)
    .append("status", "pending")
    .append("createdAt", new Date());

orders.insertOne(order);
System.out.println("Inserted ID: " + order.getObjectId("_id"));

// Insert many
orders.insertMany(Arrays.asList(
    new Document("customerId", "c124").append("total", 29.99).append("status", "pending").append("createdAt", new Date()),
    new Document("customerId", "c125").append("total", 149.00).append("status", "pending").append("createdAt", new Date())
));
```

## Find Operations

```java
import com.mongodb.client.FindIterable;
import com.mongodb.client.MongoCursor;
import com.mongodb.client.model.Filters;
import com.mongodb.client.model.Projections;
import com.mongodb.client.model.Sorts;
import org.bson.types.ObjectId;

// Find one
Document found = orders.find(Filters.eq("_id", new ObjectId("..."))).first();

// Find many with filter
FindIterable<Document> pending = orders.find(Filters.eq("status", "pending"))
    .sort(Sorts.descending("createdAt"))
    .limit(20)
    .projection(Projections.include("customerId", "total", "status"));

// Iterate
try (MongoCursor<Document> cursor = pending.iterator()) {
    while (cursor.hasNext()) {
        Document doc = cursor.next();
        System.out.println(doc.toJson());
    }
}

// Count
long count = orders.countDocuments(Filters.eq("status", "pending"));
System.out.println("Pending orders: " + count);

// Combined filters
FindIterable<Document> expensive = orders.find(
    Filters.and(
        Filters.eq("status", "pending"),
        Filters.gte("total", 100.0)
    )
);
```

## Update Operations

```java
import com.mongodb.client.model.Updates;
import com.mongodb.client.model.FindOneAndUpdateOptions;
import com.mongodb.client.model.ReturnDocument;
import com.mongodb.client.result.UpdateResult;

// Update one
UpdateResult result = orders.updateOne(
    Filters.eq("_id", new ObjectId("...")),
    Updates.combine(
        Updates.set("status", "shipped"),
        Updates.currentDate("updatedAt")
    )
);
System.out.println("Modified: " + result.getModifiedCount());

// Update many
UpdateResult bulk = orders.updateMany(
    Filters.eq("status", "pending"),
    Updates.set("flagged", true)
);

// Upsert
orders.updateOne(
    Filters.eq("externalId", "ext-001"),
    Updates.combine(
        Updates.setOnInsert("createdAt", new Date()),
        Updates.set("status", "imported")
    ),
    new com.mongodb.client.model.UpdateOptions().upsert(true)
);

// Find one and update (returns updated document)
Document updated = orders.findOneAndUpdate(
    Filters.eq("_id", new ObjectId("...")),
    Updates.set("status", "confirmed"),
    new FindOneAndUpdateOptions().returnDocument(ReturnDocument.AFTER)
);
```

## Delete Operations

```java
import com.mongodb.client.result.DeleteResult;

// Delete one
DeleteResult del = orders.deleteOne(Filters.eq("_id", new ObjectId("...")));
System.out.println("Deleted: " + del.getDeletedCount());

// Delete many
orders.deleteMany(
    Filters.and(
        Filters.eq("status", "cancelled"),
        Filters.lt("createdAt", new java.util.Date(1735689600000L))  // before 2025-01-01
    )
);
```

## Aggregation Pipeline

```java
import com.mongodb.client.model.Aggregates;
import com.mongodb.client.model.Accumulators;
import com.mongodb.client.AggregateIterable;

AggregateIterable<Document> results = orders.aggregate(Arrays.asList(
    Aggregates.match(Filters.eq("status", "completed")),
    Aggregates.group("$customerId",
        Accumulators.sum("totalSpent", "$total"),
        Accumulators.sum("orderCount", 1),
        Accumulators.avg("avgOrder", "$total")
    ),
    Aggregates.sort(Sorts.descending("totalSpent")),
    Aggregates.limit(10)
));

for (Document doc : results) {
    System.out.println(doc.toJson());
}
```

## Indexes

```java
import com.mongodb.client.model.IndexOptions;
import com.mongodb.client.model.Indexes;

// Create a compound index
orders.createIndex(Indexes.compoundIndex(
    Indexes.ascending("customerId"),
    Indexes.descending("createdAt")
));

// Unique index
db.getCollection("users").createIndex(
    Indexes.ascending("email"),
    new IndexOptions().unique(true)
);

// TTL index
db.getCollection("sessions").createIndex(
    Indexes.ascending("expiresAt"),
    new IndexOptions().expireAfter(0L, java.util.concurrent.TimeUnit.SECONDS)
);

// List indexes
db.getCollection("users").listIndexes().forEach(doc -> System.out.println(doc.toJson()));
```

## Transactions

```java
import com.mongodb.client.ClientSession;
import com.mongodb.TransactionOptions;
import com.mongodb.ReadConcern;
import com.mongodb.WriteConcern;

try (ClientSession session = client.startSession()) {
    TransactionOptions txnOptions = TransactionOptions.builder()
        .readConcern(ReadConcern.SNAPSHOT)
        .writeConcern(WriteConcern.MAJORITY)
        .build();

    session.withTransaction(() -> {
        MongoCollection<Document> accounts = db.getCollection("accounts");

        accounts.updateOne(session,
            Filters.and(Filters.eq("_id", "sender"), Filters.gte("balance", 100)),
            Updates.inc("balance", -100)
        );

        accounts.updateOne(session,
            Filters.eq("_id", "receiver"),
            Updates.inc("balance", 100)
        );

        return null;  // withTransaction requires a return value
    }, txnOptions);

    System.out.println("Transaction committed");
}
```

## POJO (Plain Old Java Object) Support

The driver supports mapping documents to Java classes using the POJO codec.

```java
import org.bson.codecs.pojo.PojoCodecProvider;
import org.bson.codecs.configuration.CodecRegistries;
import org.bson.codecs.configuration.CodecRegistry;
import com.mongodb.MongoClientSettings;

// Register POJO codec
CodecRegistry pojoCodecRegistry = CodecRegistries.fromRegistries(
    MongoClientSettings.getDefaultCodecRegistry(),
    CodecRegistries.fromProviders(PojoCodecProvider.builder().automatic(true).build())
);

MongoDatabase db = client.getDatabase("myapp").withCodecRegistry(pojoCodecRegistry);
MongoCollection<Order> orders = db.getCollection("orders", Order.class);
```

Define the Order class:

```java
public class Order {
    private ObjectId id;
    private String customerId;
    private double total;
    private String status;
    private Date createdAt;

    // Getters and setters...
}
```

```java
// Insert POJO
Order order = new Order();
order.setCustomerId("c123");
order.setTotal(99.98);
order.setStatus("pending");
order.setCreatedAt(new Date());

orders.insertOne(order);

// Find as POJO
Order found = orders.find(Filters.eq("customerId", "c123")).first();
System.out.println(found.getTotal());
```

## Error Handling

```java
import com.mongodb.MongoWriteException;
import com.mongodb.MongoCommandException;

try {
    db.getCollection("users").insertOne(
        new Document("email", "duplicate@example.com")
    );
} catch (MongoWriteException e) {
    if (e.getCode() == 11000) {
        System.err.println("Duplicate key: " + e.getError().getMessage());
    } else {
        throw e;
    }
} catch (MongoCommandException e) {
    System.err.println("Command failed: " + e.getErrorMessage());
    throw e;
}
```

## Best Practices

- Create one `MongoClient` instance per application and close it on shutdown with `client.close()`.
- Use `Filters`, `Updates`, `Sorts`, `Projections`, and `Aggregates` builder classes instead of raw Document BSON for type safety.
- Always close cursors in a try-with-resources block to prevent connection leaks.
- Use POJO support to avoid manual `Document.get()` casting.
- Configure connection pool size with `applyToConnectionPoolSettings`.
- Handle `MongoWriteException` with code `11000` for duplicate key errors.

## Summary

The MongoDB Java driver provides a comprehensive, type-safe API via builder classes like `Filters`, `Updates`, and `Aggregates`. Create a single `MongoClient` instance with configured pool settings, use POJO mapping for cleaner code, and always close cursors with try-with-resources. Transactions follow the `session.withTransaction()` pattern with `TransactionOptions` for read/write concern configuration.
