# How to Cache ASP.NET Core API Responses with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ASP.NET Core, Caching, Distributed Cache, C#

Description: Cache ASP.NET Core API responses in Redis using IDistributedCache and custom middleware to reduce database load and improve API performance.

---

## Introduction

ASP.NET Core provides the `IDistributedCache` abstraction that can be backed by Redis. You can use it directly in controllers, inject it into services, or build middleware that caches entire response bodies. This guide covers all three approaches with practical examples.

## Installation

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

## Configuration

In `Program.cs`:

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyApp:";
});
```

In `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379,abortConnect=false"
  }
}
```

## Using IDistributedCache in Controllers

```csharp
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IDistributedCache _cache;
    private readonly ProductRepository _repo;

    public ProductsController(IDistributedCache cache, ProductRepository repo)
    {
        _cache = cache;
        _repo = repo;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id)
    {
        var cacheKey = $"product:{id}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached != null)
        {
            Response.Headers["X-Cache"] = "HIT";
            return Ok(JsonSerializer.Deserialize<Product>(cached));
        }

        var product = await _repo.FindByIdAsync(id);
        if (product == null) return NotFound();

        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
            SlidingExpiration = TimeSpan.FromMinutes(2),
        };

        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(product), options);
        Response.Headers["X-Cache"] = "MISS";
        return Ok(product);
    }

    [HttpGet]
    public async Task<IActionResult> GetAll([FromQuery] int page = 1)
    {
        var cacheKey = $"products:page:{page}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached != null)
            return Ok(JsonSerializer.Deserialize<List<Product>>(cached));

        var products = await _repo.GetPageAsync(page, pageSize: 20);
        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(products),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            });

        return Ok(products);
    }
}
```

## Generic Cache Service

Create a reusable cache service:

```csharp
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

public class RedisCacheService
{
    private readonly IDistributedCache _cache;

    public RedisCacheService(IDistributedCache cache) => _cache = cache;

    public async Task<T?> GetAsync<T>(string key)
    {
        var data = await _cache.GetStringAsync(key);
        return data == null ? default : JsonSerializer.Deserialize<T>(data);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? ttl = null)
    {
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = ttl ?? TimeSpan.FromMinutes(30)
        };
        await _cache.SetStringAsync(key, JsonSerializer.Serialize(value), options);
    }

    public async Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> factory, TimeSpan? ttl = null)
    {
        var cached = await GetAsync<T>(key);
        if (cached != null) return cached;

        var value = await factory();
        await SetAsync(key, value, ttl);
        return value;
    }

    public async Task RemoveAsync(string key) => await _cache.RemoveAsync(key);
}
```

## Response Caching Middleware

```csharp
public class ResponseCacheMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IDistributedCache _cache;
    private readonly int _ttlSeconds;

    public ResponseCacheMiddleware(RequestDelegate next, IDistributedCache cache, int ttlSeconds = 300)
    {
        _next = next;
        _cache = cache;
        _ttlSeconds = ttlSeconds;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Method != "GET")
        {
            await _next(context);
            return;
        }

        var key = $"resp:{context.Request.Path}{context.Request.QueryString}";
        var cached = await _cache.GetStringAsync(key);

        if (cached != null)
        {
            context.Response.ContentType = "application/json";
            context.Response.Headers["X-Cache"] = "HIT";
            await context.Response.WriteAsync(cached);
            return;
        }

        var originalBody = context.Response.Body;
        using var buffer = new MemoryStream();
        context.Response.Body = buffer;

        await _next(context);

        buffer.Position = 0;
        var body = await new StreamReader(buffer).ReadToEndAsync();

        if (context.Response.StatusCode == 200)
        {
            await _cache.SetStringAsync(key, body, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(_ttlSeconds)
            });
        }

        buffer.Position = 0;
        context.Response.Headers["X-Cache"] = "MISS";
        context.Response.Body = originalBody;
        await buffer.CopyToAsync(originalBody);
    }
}
```

## Summary

ASP.NET Core's `IDistributedCache` interface backed by Redis provides a clean, testable way to cache API responses. Using `GetStringAsync` and `SetStringAsync` with `DistributedCacheEntryOptions` gives control over absolute and sliding expiration. A generic `RedisCacheService` with `GetOrSetAsync` eliminates boilerplate cache-aside code in controllers and services. For blanket response caching, custom middleware intercepts the response stream before it reaches the client and stores a copy in Redis.
