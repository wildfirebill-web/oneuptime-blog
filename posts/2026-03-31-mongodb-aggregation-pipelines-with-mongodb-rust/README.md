# How to Use Aggregation Pipelines with MongoDB Rust

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Rust, Aggregation, Pipeline, BSON

Description: Learn how to build and run MongoDB aggregation pipelines in Rust using the official driver with typed results and async cursor iteration.

---

## Introduction

The MongoDB Rust driver supports the full aggregation pipeline through `collection.aggregate()`. Pipelines are expressed as `Vec<Document>` using the `doc!` macro, making them easy to compose. Results can be deserialized into typed structs or left as raw BSON documents.

## Setup

```rust
use mongodb::{Client, bson::doc};
use futures::stream::TryStreamExt;
use serde::{Deserialize, Serialize};
```

## Basic $match and $group

```rust
let orders = client.database("shop").collection::<mongodb::bson::Document>("orders");

let pipeline = vec![
    doc! { "$match": { "status": "completed" } },
    doc! { "$group": {
        "_id":   "$customer_id",
        "total": { "$sum": "$amount" },
        "count": { "$sum": 1 }
    }},
    doc! { "$sort": { "total": -1 } },
    doc! { "$limit": 10 }
];

let mut cursor = orders.aggregate(pipeline).await?;
while let Some(result) = cursor.try_next().await? {
    println!("{:?}", result);
}
```

## Deserializing into a Typed Struct

```rust
#[derive(Debug, Deserialize)]
struct CustomerSummary {
    #[serde(rename = "_id")]
    customer_id: String,
    total:       f64,
    count:       i64,
}

let col = client.database("shop").collection::<mongodb::bson::Document>("orders");
let mut cursor = col.aggregate(pipeline).await?;
while let Some(doc) = cursor.try_next().await? {
    let summary: CustomerSummary = mongodb::bson::from_document(doc)?;
    println!("{}: ${:.2} ({} orders)", summary.customer_id, summary.total, summary.count);
}
```

## $lookup Pipeline Stage

```rust
let pipeline = vec![
    doc! {
        "$lookup": {
            "from":         "customers",
            "localField":   "customer_id",
            "foreignField": "_id",
            "as":           "customer"
        }
    },
    doc! { "$unwind": "$customer" },
    doc! { "$project": {
        "amount":        1,
        "customer_name": "$customer.name"
    }}
];
```

## $unwind and $group for Array Fields

```rust
let pipeline = vec![
    doc! { "$unwind": "$items" },
    doc! { "$group": {
        "_id":      "$items.product_id",
        "quantity": { "$sum": "$items.qty" }
    }},
    doc! { "$sort": { "quantity": -1 } },
    doc! { "$limit": 5 }
];
```

## $addFields and Computed Values

```rust
let pipeline = vec![
    doc! { "$addFields": {
        "tax": { "$multiply": ["$amount", 0.08] }
    }},
    doc! { "$project": {
        "amount": 1,
        "tax":    1,
        "total":  { "$add": ["$amount", "$tax"] }
    }}
];
```

## Summary

MongoDB aggregation in Rust uses `collection.aggregate(pipeline)` where the pipeline is a `Vec<Document>` built with `doc!` macros. Results are iterated via an async cursor and can be deserialized into typed structs using `bson::from_document`. Common stages - `$match`, `$group`, `$sort`, `$lookup`, `$unwind`, and `$addFields` - work identically to the MongoDB shell syntax.
