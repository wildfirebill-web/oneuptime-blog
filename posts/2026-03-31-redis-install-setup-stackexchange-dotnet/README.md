# How to Install and Set Up StackExchange.Redis in .NET

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, DotNet, StackExchange.Redis

Description: Learn how to add StackExchange.Redis to a .NET project, configure a connection multiplexer, and verify the setup with basic Redis operations.

---

StackExchange.Redis is the recommended Redis client for .NET. It uses a single shared connection multiplexer that handles connection pooling, reconnection, and command pipelining internally.

## Install the Package

```bash
dotnet add package StackExchange.Redis
```

Or via the NuGet Package Manager:

```text
Install-Package StackExchange.Redis
```

## Basic Connection

```csharp
using StackExchange.Redis;

// Create a multiplexer (this is thread-safe and should be a singleton)
ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("localhost:6379");
IDatabase db = redis.GetDatabase();

// Basic SET and GET
db.StringSet("greeting", "hello");
string? value = db.StringGet("greeting");
Console.WriteLine(value); // hello
```

## Connection with Options

```csharp
ConfigurationOptions options = new ConfigurationOptions
{
    EndPoints = { "localhost:6379" },
    Password = "yourpassword",
    ConnectTimeout = 5000,   // milliseconds
    SyncTimeout = 3000,
    AbortOnConnectFail = false, // retry on startup instead of failing fast
    ConnectRetry = 3,
};

ConnectionMultiplexer redis = ConnectionMultiplexer.Connect(options);
```

## Register as Singleton in ASP.NET Core

```csharp
// Program.cs
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration["Redis:ConnectionString"]!));
```

`appsettings.json`:

```json
{
  "Redis": {
    "ConnectionString": "localhost:6379,password=secret,connectTimeout=5000"
  }
}
```

Inject it where needed:

```csharp
public class CacheService
{
    private readonly IDatabase _db;

    public CacheService(IConnectionMultiplexer redis)
    {
        _db = redis.GetDatabase();
    }

    public void Set(string key, string value, TimeSpan? ttl = null)
        => _db.StringSet(key, value, ttl);

    public string? Get(string key)
        => _db.StringGet(key);
}
```

## Verify Connection

```csharp
IServer server = redis.GetServer("localhost", 6379);
Console.WriteLine(server.IsConnected); // True

// Ping
TimeSpan latency = db.Ping();
Console.WriteLine($"Ping: {latency.TotalMilliseconds}ms");
```

## Async Operations

StackExchange.Redis provides async versions of all commands:

```csharp
await db.StringSetAsync("key", "value", TimeSpan.FromHours(1));
string? val = await db.StringGetAsync("key");
Console.WriteLine(val);
```

## Connection String Format

StackExchange.Redis uses a compact connection string format:

```text
localhost:6379,password=secret,ssl=false,connectTimeout=5000,syncTimeout=3000,abortConnect=false
```

## Summary

StackExchange.Redis is installed via NuGet and configured through `ConnectionMultiplexer.Connect`. The multiplexer is thread-safe and should be created once as a singleton, not per-request. In ASP.NET Core, register it with `AddSingleton` and inject `IConnectionMultiplexer` into services. All commands have synchronous and async variants, with async being preferred in web application code.
