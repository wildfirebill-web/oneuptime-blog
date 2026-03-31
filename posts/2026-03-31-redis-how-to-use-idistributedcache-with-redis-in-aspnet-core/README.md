# How to Use IDistributedCache with Redis in ASP.NET Core

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ASP.NET Core, C#, IDistributedCache, Caching

Description: Learn how to use IDistributedCache with Redis in ASP.NET Core for distributed caching, including setup, storing complex objects, and cache-aside patterns.

---

## What Is IDistributedCache?

`IDistributedCache` is the ASP.NET Core abstraction for distributed caches. Using it instead of Redis directly:
- Enables swapping cache implementations without changing business logic
- Provides a consistent API (string and byte array operations)
- Supports JSON serialization for complex objects
- Works with Redis, SQL Server, NCache, and custom implementations

## Installation

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

## Basic Configuration

In `Program.cs`:

```csharp
using Microsoft.Extensions.Caching.StackExchangeRedis;

var builder = WebApplication.CreateBuilder(args);

// Add Redis distributed cache
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "MyApp:";  // Optional key prefix
});

var app = builder.Build();
```

Or using `appsettings.json`:

```json
{
  "Redis": {
    "ConnectionString": "localhost:6379,password=yourpassword",
    "InstanceName": "MyApp:"
  }
}
```

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:ConnectionString"];
    options.InstanceName = builder.Configuration["Redis:InstanceName"];
});
```

## Basic Cache Operations

```csharp
using Microsoft.Extensions.Caching.Distributed;

public class CacheService
{
    private readonly IDistributedCache _cache;

    public CacheService(IDistributedCache cache)
    {
        _cache = cache;
    }

    // Store a string
    public async Task SetStringAsync(string key, string value, int ttlSeconds = 300)
    {
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(ttlSeconds)
        };
        await _cache.SetStringAsync(key, value, options);
    }

    // Get a string
    public async Task<string?> GetStringAsync(string key)
    {
        return await _cache.GetStringAsync(key);
    }

    // Remove an entry
    public async Task RemoveAsync(string key)
    {
        await _cache.RemoveAsync(key);
    }

    // Refresh expiry (extends sliding expiry)
    public async Task RefreshAsync(string key)
    {
        await _cache.RefreshAsync(key);
    }
}
```

## Caching Complex Objects with JSON

```csharp
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

public class UserCacheService
{
    private readonly IDistributedCache _cache;
    private static readonly JsonSerializerOptions JsonOptions = new() { PropertyNameCaseInsensitive = true };

    public UserCacheService(IDistributedCache cache)
    {
        _cache = cache;
    }

    private string UserKey(int userId) => $"user:{userId}";

    public async Task CacheUserAsync(User user, int ttlSeconds = 3600)
    {
        string json = JsonSerializer.Serialize(user, JsonOptions);
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(ttlSeconds)
        };
        await _cache.SetStringAsync(UserKey(user.Id), json, options);
    }

    public async Task<User?> GetUserAsync(int userId)
    {
        string? json = await _cache.GetStringAsync(UserKey(userId));
        if (json == null) return null;

        return JsonSerializer.Deserialize<User>(json, JsonOptions);
    }

    public async Task InvalidateUserAsync(int userId)
    {
        await _cache.RemoveAsync(UserKey(userId));
    }

    // Cache-aside pattern
    public async Task<User?> GetOrFetchAsync(int userId, Func<Task<User?>> fetcher, int ttlSeconds = 3600)
    {
        var cached = await GetUserAsync(userId);
        if (cached != null) return cached;

        var user = await fetcher();
        if (user != null)
        {
            await CacheUserAsync(user, ttlSeconds);
        }

        return user;
    }
}

public record User(int Id, string Name, string Email, string Role);
```

## Sliding Expiry vs Absolute Expiry

```csharp
using Microsoft.Extensions.Caching.Distributed;

// Absolute expiry - expires at fixed time regardless of access
var absoluteOptions = new DistributedCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
};

// Sliding expiry - resets on each access
var slidingOptions = new DistributedCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(20)
};

// Combined - sliding expiry with absolute cap
var combinedOptions = new DistributedCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(20),
    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(2)
};

await _cache.SetStringAsync("key", "value", combinedOptions);
```

## Controller Integration

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IDistributedCache _cache;
    private readonly IProductRepository _repo;

    public ProductsController(IDistributedCache cache, IProductRepository repo)
    {
        _cache = cache;
        _repo = repo;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetProduct(int id)
    {
        string cacheKey = $"product:{id}";

        // Try cache first
        string? cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            var cachedProduct = JsonSerializer.Deserialize<Product>(cached);
            return Ok(new { source = "cache", data = cachedProduct });
        }

        // Fetch from database
        var product = await _repo.GetByIdAsync(id);
        if (product == null) return NotFound();

        // Cache the result
        string json = JsonSerializer.Serialize(product);
        await _cache.SetStringAsync(cacheKey, json, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15)
        });

        return Ok(new { source = "database", data = product });
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateProduct(int id, [FromBody] Product product)
    {
        await _repo.UpdateAsync(product);

        // Invalidate cache
        await _cache.RemoveAsync($"product:{id}");

        return Ok(product);
    }
}
```

## Generic Cache Helper

```csharp
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

public class GenericCacheHelper
{
    private readonly IDistributedCache _cache;

    public GenericCacheHelper(IDistributedCache cache)
    {
        _cache = cache;
    }

    public async Task<T?> GetAsync<T>(string key)
    {
        string? json = await _cache.GetStringAsync(key);
        return json == null ? default : JsonSerializer.Deserialize<T>(json);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan ttl)
    {
        string json = JsonSerializer.Serialize(value);
        await _cache.SetStringAsync(key, json, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = ttl
        });
    }

    public async Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> factory, TimeSpan ttl)
    {
        var cached = await GetAsync<T>(key);
        if (cached != null) return cached;

        T value = await factory();
        await SetAsync(key, value, ttl);
        return value;
    }
}
```

## Summary

`IDistributedCache` with Redis in ASP.NET Core provides a clean, swappable caching abstraction via `AddStackExchangeRedisCache()`. Use `SetStringAsync`/`GetStringAsync` with JSON serialization for complex objects, configure absolute or sliding expiry via `DistributedCacheEntryOptions`, and implement the cache-aside pattern with `GetOrSetAsync` helpers. Inject `IDistributedCache` into controllers and services rather than using Redis directly to maintain abstraction and testability.
