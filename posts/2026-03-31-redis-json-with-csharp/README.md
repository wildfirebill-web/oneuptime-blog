# How to Use Redis JSON with C#

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisJSON, CSharp, .NET, JSON

Description: Learn how to store, retrieve, and manipulate JSON documents in Redis using RedisJSON and the NRedisStack C# client.

---

RedisJSON is a Redis module that lets you store and query JSON documents natively. With NRedisStack, C# developers get a full-featured typed API for all JSON operations.

## Setup

```bash
dotnet add package NRedisStack
```

```bash
docker run -d -p 6379:6379 redis/redis-stack-server:latest
```

## Connecting

```csharp
using NRedisStack;
using NRedisStack.RedisStackCommands;
using StackExchange.Redis;

var mux = ConnectionMultiplexer.Connect("localhost:6379");
var db = mux.GetDatabase();
var json = db.JSON();
```

## Storing a JSON Document

Use `JSON.SET` to store any serializable C# object:

```csharp
var product = new
{
    id = 101,
    name = "Mechanical Keyboard",
    price = 89.99,
    tags = new[] { "hardware", "input" },
    stock = 50
};

await json.SetAsync("product:101", "$", product);
```

## Retrieving a Document

```csharp
// Full document
var result = await json.GetAsync<dynamic>("product:101", path: "$");

// Specific field using JSONPath
var price = await json.GetAsync<double>("product:101", path: "$.price");
Console.WriteLine($"Price: {price}");
```

## Updating Fields

```csharp
// Update the price
await json.SetAsync("product:101", "$.price", 79.99);

// Decrement stock by 1
await json.NumIncrbyAsync("product:101", "$.stock", -1);
```

## Working with Arrays

```csharp
// Append a tag
await json.ArrAppendAsync("product:101", "$.tags", "mechanical");

// Get array length
var len = await json.ArrLenAsync("product:101", "$.tags");
Console.WriteLine($"Tags count: {len[0]}");
```

## Getting Keys and Types

```csharp
// Object keys
var keys = await json.ObjKeysAsync("product:101", "$");
// Returns: id, name, price, tags, stock

// Type of a field
var type = await json.TypeAsync("product:101", "$.tags");
Console.WriteLine(type[0]); // array
```

## Deleting Fields

```csharp
// Remove a field
await json.DelAsync("product:101", "$.stock");

// Delete the whole document
await json.DelAsync("product:101");
```

## Retrieving Multiple Keys with MGET

```csharp
await json.SetAsync("product:102", "$", new { id = 102, name = "Mouse", price = 29.99 });

var items = await json.MGetAsync(
    new RedisKey[] { "product:101", "product:102" },
    path: "$.name");

foreach (var item in items)
    Console.WriteLine(item);
```

## Strongly Typed Deserialization

```csharp
public record Product(int Id, string Name, double Price);

var product = await json.GetAsync<Product>("product:101", path: "$");
Console.WriteLine(product?.Name);
```

RedisJSON with NRedisStack is an efficient replacement for storing serialized blobs in plain Redis strings. JSONPath expressions let you update deeply nested fields without reading and rewriting entire documents.

## Summary

NRedisStack's JSON commands give C# applications a clean way to store and manipulate structured data in Redis. JSONPath support enables partial updates and targeted reads, making it ideal for product catalogs, user profiles, and configuration storage.
