# How to Perform CRUD Operations with the MongoDB .NET Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, CSharp, DotNet, CRUD, Driver

Description: Learn how to insert, find, update, and delete documents in MongoDB using the official .NET Driver with async C# examples.

---

## Overview

The MongoDB .NET Driver provides an async-first API for all CRUD operations. Collections are typed - `IMongoCollection<T>` - so your C# model classes map directly to MongoDB documents. This guide walks through all four operation types with practical examples.

## Setup

```bash
dotnet add package MongoDB.Driver
```

Document model:

```csharp
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;

public class Product
{
    [BsonId]
    [BsonRepresentation(BsonType.ObjectId)]
    public string Id { get; set; }
    public string Name { get; set; }
    public double Price { get; set; }
    public string Category { get; set; }
    public int Stock { get; set; }
}
```

Getting a collection:

```csharp
var client = new MongoClient("mongodb://localhost:27017");
var db = client.GetDatabase("shopdb");
var products = db.GetCollection<Product>("products");
```

## Create - Insert Documents

```csharp
// Insert one document
var keyboard = new Product
{
    Name = "Wireless Keyboard",
    Price = 49.99,
    Category = "electronics",
    Stock = 100
};
await products.InsertOneAsync(keyboard);
Console.WriteLine($"Inserted: {keyboard.Id}");

// Insert many documents
var newProducts = new List<Product>
{
    new Product { Name = "Mouse", Price = 29.99, Category = "electronics", Stock = 200 },
    new Product { Name = "Monitor", Price = 299.99, Category = "electronics", Stock = 50 }
};
await products.InsertManyAsync(newProducts);
```

## Read - Find Documents

```csharp
using MongoDB.Driver;

// Find all
var all = await products.Find(_ => true).ToListAsync();

// Find with filter builder
var filter = Builders<Product>.Filter.Eq(p => p.Category, "electronics");
var electronics = await products.Find(filter).ToListAsync();

// Find with multiple conditions
var affordable = await products.Find(
    Builders<Product>.Filter.And(
        Builders<Product>.Filter.Eq(p => p.Category, "electronics"),
        Builders<Product>.Filter.Lte(p => p.Price, 100.0),
        Builders<Product>.Filter.Gt(p => p.Stock, 0)
    )
).ToListAsync();

// Find one
var keyboard = await products
    .Find(p => p.Name == "Wireless Keyboard")
    .FirstOrDefaultAsync();

// Find by ID
var byId = await products
    .Find(p => p.Id == "507f1f77bcf86cd799439011")
    .SingleOrDefaultAsync();
```

## Sorting and Projection

```csharp
// Sort and limit
var topProducts = await products
    .Find(p => p.Category == "electronics")
    .SortBy(p => p.Price)
    .Limit(10)
    .ToListAsync();

// Projection - return only Name and Price
var projected = await products
    .Find(_ => true)
    .Project(Builders<Product>.Projection
        .Include(p => p.Name)
        .Include(p => p.Price)
        .Exclude(p => p.Id))
    .ToListAsync();
```

## Update - Modify Documents

```csharp
// UpdateOne
var updateFilter = Builders<Product>.Filter.Eq(p => p.Name, "Wireless Keyboard");
var update = Builders<Product>.Update
    .Set(p => p.Price, 44.99)
    .Inc(p => p.Stock, -1)
    .CurrentDate("lastModified");

var updateResult = await products.UpdateOneAsync(updateFilter, update);
Console.WriteLine($"Modified: {updateResult.ModifiedCount}");

// UpdateMany
await products.UpdateManyAsync(
    Builders<Product>.Filter.Eq(p => p.Stock, 0),
    Builders<Product>.Update.Set(p => p.Category, "out-of-stock")
);

// ReplaceOne - replace the entire document
var replacement = new Product
{
    Id = keyboard.Id,
    Name = "Wireless Keyboard Pro",
    Price = 59.99,
    Category = "electronics",
    Stock = 75
};
await products.ReplaceOneAsync(p => p.Id == keyboard.Id, replacement);
```

## Upsert

```csharp
var upsertResult = await products.UpdateOneAsync(
    Builders<Product>.Filter.Eq(p => p.Name, "New Product"),
    Builders<Product>.Update
        .SetOnInsert(p => p.Name, "New Product")
        .Set(p => p.Price, 19.99)
        .Set(p => p.Stock, 50),
    new UpdateOptions { IsUpsert = true }
);
Console.WriteLine($"Upserted ID: {upsertResult.UpsertedId}");
```

## Delete - Remove Documents

```csharp
// DeleteOne
var deleteResult = await products.DeleteOneAsync(p => p.Name == "Wireless Keyboard");
Console.WriteLine($"Deleted: {deleteResult.DeletedCount}");

// DeleteMany
await products.DeleteManyAsync(p => p.Category == "discontinued");
```

## FindOneAndUpdate (Atomic)

```csharp
var updated = await products.FindOneAndUpdateAsync(
    p => p.Name == "Wireless Keyboard",
    Builders<Product>.Update.Inc(p => p.Stock, -1),
    new FindOneAndUpdateOptions<Product> { ReturnDocument = ReturnDocument.After }
);
Console.WriteLine($"New stock: {updated.Stock}");
```

## Summary

The MongoDB .NET Driver provides a clean async API for all CRUD operations. Use `InsertOneAsync`/`InsertManyAsync` for creates, `Find` with `Builders<T>.Filter` for reads, `UpdateOneAsync`/`UpdateManyAsync` with `Builders<T>.Update` for updates, and `DeleteOneAsync`/`DeleteManyAsync` for deletes. Always use `FindOneAndUpdateAsync` when you need atomic read-modify operations to avoid race conditions.
