# How to Use Redis Hashes in C#

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CSharp, Hash

Description: Learn how to store, retrieve, update, and delete Redis hash fields in C# using StackExchange.Redis with practical object storage examples.

---

Redis hashes map string field-value pairs under a single key. They are ideal for storing structured objects like user profiles, product records, and configuration. StackExchange.Redis exposes hash operations through `HashSet`, `HashGet`, `HashGetAll`, and related methods.

## Set and Get Hash Fields

```csharp
using StackExchange.Redis;

IDatabase db = redis.GetDatabase();

// Set multiple fields
await db.HashSetAsync("user:1", new HashEntry[]
{
    new HashEntry("name", "Alice"),
    new HashEntry("email", "alice@example.com"),
    new HashEntry("age", "30"),
    new HashEntry("role", "admin"),
});

// Get a single field
RedisValue name = await db.HashGetAsync("user:1", "name");
Console.WriteLine(name); // Alice

// Get multiple fields
RedisValue[] vals = await db.HashGetAsync("user:1",
    new RedisValue[] { "name", "email" });
Console.WriteLine($"{vals[0]}, {vals[1]}"); // Alice, alice@example.com
```

## Get All Fields

```csharp
HashEntry[] all = await db.HashGetAllAsync("user:1");
foreach (HashEntry entry in all)
{
    Console.WriteLine($"{entry.Name}: {entry.Value}");
}
```

## Convert to Dictionary

```csharp
HashEntry[] entries = await db.HashGetAllAsync("user:1");
Dictionary<string, string> dict = entries.ToDictionary(
    e => e.Name.ToString(),
    e => e.Value.ToString());
```

## Increment a Numeric Field

```csharp
long newAge = await db.HashIncrementAsync("user:1", "age", 1);
Console.WriteLine(newAge); // 31

double newScore = await db.HashIncrementAsync("user:1", "score", 0.5);
```

## Conditional Set

```csharp
// Only set if field does not exist
bool created = await db.HashSetAsync("user:1", "verified", "false",
    When.NotExists);
Console.WriteLine(created); // true if field was created, false if it existed
```

## Delete Fields

```csharp
bool deleted = await db.HashDeleteAsync("user:1", "role");
Console.WriteLine(deleted); // true if field existed and was deleted

// Delete multiple fields
long count = await db.HashDeleteAsync("user:1",
    new RedisValue[] { "verified", "role" });
```

## Check Field Existence

```csharp
bool exists = await db.HashExistsAsync("user:1", "email");
Console.WriteLine(exists); // true
```

## List Keys and Count

```csharp
RedisValue[] keys = await db.HashKeysAsync("user:1");
RedisValue[] values = await db.HashValuesAsync("user:1");
long fieldCount = await db.HashLengthAsync("user:1");
```

## Storing a POCO as a Hash

```csharp
public record Product(int Id, string Name, decimal Price, string Category);

async Task StoreProduct(IDatabase db, Product p)
{
    await db.HashSetAsync($"product:{p.Id}", new HashEntry[]
    {
        new HashEntry("name", p.Name),
        new HashEntry("price", p.Price.ToString("F2")),
        new HashEntry("category", p.Category),
    });
}

async Task<Product?> GetProduct(IDatabase db, int id)
{
    HashEntry[] fields = await db.HashGetAllAsync($"product:{id}");
    if (fields.Length == 0) return null;

    var d = fields.ToDictionary(e => e.Name.ToString(), e => e.Value.ToString());
    return new Product(id, d["name"], decimal.Parse(d["price"]), d["category"]);
}
```

## Summary

Redis hashes in C# use `HashSetAsync` with `HashEntry` arrays to set multiple fields at once. `HashGetAllAsync` returns all fields, which can be converted to a dictionary. `HashIncrementAsync` atomically increments numeric fields. Storing structured objects as hashes is more memory-efficient than individual string keys and supports partial field updates without rewriting the whole object.
