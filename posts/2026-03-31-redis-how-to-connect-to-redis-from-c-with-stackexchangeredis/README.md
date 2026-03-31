# How to Connect to Redis from C# with StackExchange.Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, C#, .NET, StackExchange.Redis, Connection

Description: Learn how to connect to Redis from C# using StackExchange.Redis, configure ConnectionMultiplexer, handle authentication, and perform common operations.

---

## What Is StackExchange.Redis

StackExchange.Redis is the primary Redis client library for .NET, developed and maintained by Stack Overflow. It uses a single, thread-safe `ConnectionMultiplexer` that multiplexes all Redis operations over a small number of connections, making it highly efficient for ASP.NET applications.

NuGet: https://www.nuget.org/packages/StackExchange.Redis

## Installing the Package

```bash
# .NET CLI
dotnet add package StackExchange.Redis

# Package Manager Console
Install-Package StackExchange.Redis
```

## Basic Connection

```csharp
using StackExchange.Redis;

// Create a single multiplexer (reuse this across the application)
ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("localhost:6379");

// Get a database handle
IDatabase db = redis.GetDatabase();

// Basic string operations
await db.StringSetAsync("user:1001", "Alice");
RedisValue value = await db.StringGetAsync("user:1001");
Console.WriteLine($"Got: {value}"); // Alice

// Set with expiration
await db.StringSetAsync("session:abc123", "user_data", TimeSpan.FromHours(1));
TimeSpan? ttl = await db.KeyTimeToLiveAsync("session:abc123");
Console.WriteLine($"TTL: {ttl?.TotalSeconds}s");

redis.Dispose();
```

## Configuration Options

```csharp
ConfigurationOptions options = new ConfigurationOptions
{
    EndPoints = { "redis.production.example.com:6379" },
    Password = "your-redis-password",
    ConnectTimeout = 5000,       // milliseconds
    SyncTimeout = 5000,
    AsyncTimeout = 5000,
    ConnectRetry = 3,
    AbortOnConnectFail = false,  // Don't throw on initial connect failure
    AllowAdmin = false,          // Disable admin commands (FLUSHALL, DEBUG, etc.)
    Ssl = true,                  // Enable TLS
    SslProtocols = System.Security.Authentication.SslProtocols.Tls12,
    DefaultDatabase = 0
};

ConnectionMultiplexer redis = await ConnectionMultiplexer.ConnectAsync(options);
```

## Dependency Injection in ASP.NET Core

The recommended way to use StackExchange.Redis in ASP.NET Core is via the `IConnectionMultiplexer` abstraction:

```csharp
// Program.cs
using StackExchange.Redis;

var builder = WebApplication.CreateBuilder(args);

// Register as singleton - ConnectionMultiplexer is thread-safe and expensive to create
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")!));

// Or use the Microsoft.Extensions.Caching.StackExchangeRedis package for IDistributedCache
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyApp:";
});

var app = builder.Build();
```

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379,password=yourpassword,abortConnect=false"
  }
}
```

## Working with Data Structures

```csharp
using StackExchange.Redis;

IDatabase db = redis.GetDatabase();

// Hashes
await db.HashSetAsync("user:1001", new HashEntry[]
{
    new("name", "Alice"),
    new("email", "alice@example.com"),
    new("age", 30)
});

RedisValue name = await db.HashGetAsync("user:1001", "name");
HashEntry[] allFields = await db.HashGetAllAsync("user:1001");
Console.WriteLine($"Name: {name}");

// Lists
await db.ListRightPushAsync("tasks", new RedisValue[] { "task1", "task2", "task3" });
RedisValue task = await db.ListLeftPopAsync("tasks");
long count = await db.ListLengthAsync("tasks");
Console.WriteLine($"Popped: {task}, remaining: {count}");

// Sets
await db.SetAddAsync("active_users", new RedisValue[] { "alice", "bob", "charlie" });
bool isMember = await db.SetContainsAsync("active_users", "alice");
RedisValue[] members = await db.SetMembersAsync("active_users");
Console.WriteLine($"Alice is active: {isMember}");

// Sorted Sets
await db.SortedSetAddAsync("leaderboard", new SortedSetEntry[]
{
    new("alice", 1500),
    new("bob", 2200),
    new("charlie", 1800)
});

RedisValue[] top3 = await db.SortedSetRangeByRankAsync("leaderboard", 0, 2, Order.Descending);
Console.WriteLine($"Top 3: {string.Join(", ", top3)}");
```

## Batching with CreateBatch

```csharp
IBatch batch = db.CreateBatch();

// Queue operations without waiting for individual results
Task<bool> setTask1 = batch.StringSetAsync("key:1", "value1");
Task<bool> setTask2 = batch.StringSetAsync("key:2", "value2");
Task<RedisValue> getTask = batch.StringGetAsync("key:1");

// Execute all at once
batch.Execute();

// Await results after Execute()
bool set1 = await setTask1;
RedisValue result = await getTask;
Console.WriteLine($"Set: {set1}, Got: {result}");
```

## Transactions with CreateTransaction

```csharp
ITransaction tran = db.CreateTransaction();

// Add condition (WATCH equivalent)
tran.AddCondition(Condition.KeyExists("lock:resource1"));

// Queue commands
Task<bool> setTask = tran.StringSetAsync("counter", 42);
Task<long> incrTask = tran.StringIncrementAsync("counter");

// Execute - returns false if condition fails
bool committed = await tran.ExecuteAsync();
if (committed)
{
    long newValue = await incrTask;
    Console.WriteLine($"New counter value: {newValue}");
}
else
{
    Console.WriteLine("Transaction aborted - condition not met");
}
```

## Pub/Sub

```csharp
ISubscriber sub = redis.GetSubscriber();

// Subscribe to a channel
await sub.SubscribeAsync(RedisChannel.Literal("notifications"), (channel, message) =>
{
    Console.WriteLine($"Received on {channel}: {message}");
});

// Publish from another part of the application
ISubscriber pub = redis.GetSubscriber();
await pub.PublishAsync(RedisChannel.Literal("notifications"), "Hello subscribers!");
```

## Handling Connection Events

```csharp
redis.ConnectionFailed += (sender, args) =>
{
    Console.Error.WriteLine($"Connection failed: {args.Exception?.Message}");
};

redis.ConnectionRestored += (sender, args) =>
{
    Console.WriteLine("Connection restored");
};

redis.ErrorMessage += (sender, args) =>
{
    Console.Error.WriteLine($"Redis error: {args.Message}");
};
```

## Summary

StackExchange.Redis is the standard Redis client for .NET applications. Register `ConnectionMultiplexer` as a singleton in ASP.NET Core's DI container - it handles all multiplexing internally and is safe to share. Use `IDatabase` for standard operations, `CreateBatch` for pipelining, and `CreateTransaction` with `AddCondition` for optimistic locking. The async API (`StringSetAsync`, `HashGetAsync`, etc.) integrates seamlessly with `async/await` patterns in modern .NET.
