# How to Use Redis for ASP.NET Core Session State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ASP.NET Core, Session, Cache, C#

Description: Configure Redis-backed session state in ASP.NET Core to share user sessions across multiple instances without sticky sessions.

---

By default, ASP.NET Core stores session data in memory, which means sessions are lost on restart and don't work across multiple server instances. Switching to Redis gives you persistent, distributed session storage.

## Install Required Packages

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
dotnet add package Microsoft.AspNetCore.Session
```

## Configure Services in Program.cs

Register both the Redis distributed cache and the session middleware:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register Redis as the distributed cache backend
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "session:";
});

// Register session services
builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

var app = builder.Build();

app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.UseSession(); // Must come before MapControllers
app.MapControllers();
app.Run();
```

The `IdleTimeout` controls how long a session stays alive after the last request. Each access resets the timer.

## Read and Write Session Values

ASP.NET Core session stores data as byte arrays. Use the extension methods `SetString`, `GetString`, `SetInt32`, and `GetInt32` for common types:

```csharp
[ApiController]
[Route("api/[controller]")]
public class CartController : ControllerBase
{
    [HttpPost("add")]
    public IActionResult AddItem([FromBody] CartItem item)
    {
        var cartJson = HttpContext.Session.GetString("cart") ?? "[]";
        var cart = JsonSerializer.Deserialize<List<CartItem>>(cartJson)!;
        cart.Add(item);
        HttpContext.Session.SetString("cart", JsonSerializer.Serialize(cart));
        return Ok(new { itemCount = cart.Count });
    }

    [HttpGet]
    public IActionResult GetCart()
    {
        var cartJson = HttpContext.Session.GetString("cart") ?? "[]";
        var cart = JsonSerializer.Deserialize<List<CartItem>>(cartJson);
        return Ok(cart);
    }

    [HttpDelete]
    public IActionResult ClearCart()
    {
        HttpContext.Session.Remove("cart");
        return NoContent();
    }
}
```

## Store Complex Objects

Serialize complex types to JSON before storing them in the session:

```csharp
public static class SessionExtensions
{
    public static void Set<T>(this ISession session, string key, T value)
    {
        var json = JsonSerializer.Serialize(value);
        session.SetString(key, json);
    }

    public static T? Get<T>(this ISession session, string key)
    {
        var json = session.GetString(key);
        return json is null ? default : JsonSerializer.Deserialize<T>(json);
    }
}
```

## Verify Session Keys in Redis

After storing a session, confirm the key exists in Redis:

```bash
redis-cli keys "session:*"
# Example output: session:abc123...
redis-cli ttl "session:abc123..."
# Returns remaining seconds
```

## Summary

Redis-backed sessions in ASP.NET Core require two packages and a few lines in `Program.cs`. Once configured, the `HttpContext.Session` API works identically to in-memory sessions, but data is stored in Redis and shared across all app instances. This eliminates the need for sticky sessions at the load balancer and keeps sessions alive across app restarts.
