# How to Configure ASP.NET Core Distributed Cache with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ASP.NET Core, Cache, Distributed System, C#

Description: Step-by-step guide to setting up IDistributedCache with Redis in ASP.NET Core for scalable, multi-instance caching.

---

ASP.NET Core's `IDistributedCache` interface provides a unified caching API that works with multiple backends. Redis is the most popular production choice because it is fast, supports TTLs natively, and works across multiple app instances.

## Install the NuGet Package

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

## Register the Redis Cache in Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "myapp:";
});
```

Add the connection string to `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379,abortConnect=false"
  }
}
```

The `InstanceName` prefix is prepended to every key, which prevents collisions between apps sharing the same Redis instance.

## Use IDistributedCache in a Controller

Inject `IDistributedCache` and use `GetStringAsync` / `SetStringAsync` for simple string values:

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IDistributedCache _cache;
    private readonly ProductService _productService;

    public ProductsController(IDistributedCache cache, ProductService productService)
    {
        _cache = cache;
        _productService = productService;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetProduct(int id)
    {
        var cacheKey = $"product:{id}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached is not null)
        {
            var product = JsonSerializer.Deserialize<Product>(cached);
            return Ok(product);
        }

        var freshProduct = await _productService.GetByIdAsync(id);
        if (freshProduct is null) return NotFound();

        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
            SlidingExpiration = TimeSpan.FromMinutes(2)
        };

        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(freshProduct), options);
        return Ok(freshProduct);
    }
}
```

## Cache Binary Data

For non-string values, use `GetAsync` and `SetAsync` with a byte array:

```csharp
byte[] imageBytes = await _storage.GetImageAsync(imageId);

await _cache.SetAsync($"image:{imageId}", imageBytes, new DistributedCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
});
```

## Invalidate Cache Entries

Remove a specific entry when the underlying data changes:

```csharp
await _cache.RemoveAsync($"product:{id}");
```

## Use Connection Resilience

Enable retry logic in the Redis connection string to handle transient failures:

```json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379,abortConnect=false,connectRetry=3,connectTimeout=5000"
  }
}
```

## Summary

Configuring `IDistributedCache` with Redis in ASP.NET Core takes just one NuGet package and a few lines in `Program.cs`. Use `GetStringAsync`/`SetStringAsync` for JSON-serialized objects and `DistributedCacheEntryOptions` to control expiration. This pattern scales across multiple app instances with no code changes since all instances share the same Redis backend.
