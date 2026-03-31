# How to Use Aggregation Pipelines with the MongoDB Java Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Java, Aggregation, Pipeline, Driver

Description: Learn how to build and execute MongoDB aggregation pipelines with the Java driver using the Aggregates builder for type-safe pipeline construction.

---

## Overview

The MongoDB Java driver provides an `Aggregates` builder class that constructs pipeline stages with a fluent, type-safe API. Pipelines are passed to `collection.aggregate()`, which returns an `AggregateIterable` cursor for iterating results.

## Setup

```xml
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver-sync</artifactId>
    <version>5.1.0</version>
</dependency>
```

```java
import com.mongodb.client.*;
import com.mongodb.client.model.*;
import org.bson.Document;
import java.util.*;
import static com.mongodb.client.model.Aggregates.*;
import static com.mongodb.client.model.Accumulators.*;
import static com.mongodb.client.model.Filters.*;
import static com.mongodb.client.model.Sorts.*;

MongoClient client = MongoClients.create("mongodb://localhost:27017");
MongoCollection<Document> col = client.getDatabase("shop").getCollection("orders");
```

## Basic Pipeline

```java
List<Document> results = col.aggregate(List.of(
    match(eq("status", "completed")),
    group("$customerId",
        sum("total", "$amount"),
        sum("count", 1L)
    ),
    sort(descending("total")),
    limit(10)
)).into(new ArrayList<>());

for (Document doc : results) {
    System.out.println(doc.toJson());
}
```

## Projection Stage

```java
List<Document> orders = col.aggregate(List.of(
    match(gte("amount", 100.0)),
    project(Projections.fields(
        Projections.include("customerId", "amount", "status"),
        Projections.excludeId(),
        Projections.computed("amountCents",
            new Document("$multiply", List.of("$amount", 100)))
    ))
)).into(new ArrayList<>());
```

## Lookup (Join) Stage

```java
List<Document> enriched = col.aggregate(List.of(
    match(eq("status", "pending")),
    lookup("customers", "customerId", "_id", "customerInfo"),
    unwind("$customerInfo"),
    project(Projections.fields(
        Projections.include("amount", "status"),
        Projections.computed("customerName", "$customerInfo.name"),
        Projections.computed("customerEmail", "$customerInfo.email"),
        Projections.excludeId()
    ))
)).into(new ArrayList<>());
```

## Unwind Stage

```java
// Unwind tags array and count each tag
List<Document> tagCounts = col.aggregate(List.of(
    unwind("$tags"),
    group("$tags", sum("count", 1L)),
    sort(descending("count")),
    limit(20)
)).into(new ArrayList<>());
```

## addFields Stage

```java
List<Document> withDiscount = col.aggregate(List.of(
    addFields(new Field<>("discountedPrice",
        new Document("$multiply", List.of("$price", 0.9))
    ))
)).into(new ArrayList<>());
```

## Allowing Disk Use

For pipelines that exceed the 100 MB memory limit:

```java
AggregateIterable<Document> cursor = col.aggregate(pipeline)
    .allowDiskUse(true)
    .batchSize(500);

for (Document doc : cursor) {
    process(doc);
}
```

## Using $facet for Multi-Dimensional Analytics

```java
List<Document> facetResult = col.aggregate(List.of(
    facet(
        new Facet("byCategory",
            group("$category", sum("total", "$amount"))
        ),
        new Facet("topOrders",
            sort(descending("amount")),
            limit(5)
        )
    )
)).into(new ArrayList<>());
```

## Summary

The MongoDB Java driver's `Aggregates` builder provides type-safe methods for `match`, `group`, `project`, `lookup`, `unwind`, `sort`, `limit`, `addFields`, and `facet`. Run pipelines with `collection.aggregate(pipeline)` and iterate with `into()` or a for-each loop. Use `.allowDiskUse(true)` for large pipelines and `.batchSize()` to control cursor batching for memory-efficient result processing.
