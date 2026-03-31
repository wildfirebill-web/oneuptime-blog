# How to Implement a Shopping Cart with Dapr Actor State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, State Management, E-Commerce, Microservice

Description: Build a shopping cart service using Dapr virtual actors to manage per-user cart state with automatic persistence, concurrency control, and TTL-based expiry.

---

## Overview

A shopping cart is a classic use case for the virtual actor pattern. Each user's cart maps to a dedicated actor instance, giving you automatic single-writer concurrency, built-in state persistence, and natural TTL management through actor idle timeout.

## Defining the Cart Actor Interface

```csharp
// .NET SDK actor interface
public interface ICartActor : IActor
{
    Task AddItemAsync(CartItem item);
    Task RemoveItemAsync(string productId);
    Task<List<CartItem>> GetItemsAsync();
    Task ClearAsync();
    Task<decimal> GetTotalAsync();
}
```

## Implementing the Cart Actor

```csharp
public class CartActor : Actor, ICartActor
{
    private const string CartStateKey = "cart-items";

    public CartActor(ActorHost host) : base(host) { }

    public async Task AddItemAsync(CartItem item)
    {
        var items = await GetItemsAsync();
        var existing = items.FirstOrDefault(i => i.ProductId == item.ProductId);
        if (existing != null)
            existing.Quantity += item.Quantity;
        else
            items.Add(item);

        await StateManager.SetStateAsync(CartStateKey, items);
    }

    public async Task RemoveItemAsync(string productId)
    {
        var items = await GetItemsAsync();
        items.RemoveAll(i => i.ProductId == productId);
        await StateManager.SetStateAsync(CartStateKey, items);
    }

    public async Task<List<CartItem>> GetItemsAsync()
    {
        var result = await StateManager.TryGetStateAsync<List<CartItem>>(CartStateKey);
        return result.HasValue ? result.Value : new List<CartItem>();
    }

    public async Task<decimal> GetTotalAsync()
    {
        var items = await GetItemsAsync();
        return items.Sum(i => i.Price * i.Quantity);
    }

    public async Task ClearAsync()
    {
        await StateManager.RemoveStateAsync(CartStateKey);
    }
}
```

## Calling the Cart Actor from an API

```csharp
// ASP.NET Core controller using Dapr actor proxy
[HttpPost("/cart/{userId}/items")]
public async Task<IActionResult> AddItem(string userId, [FromBody] CartItem item)
{
    var proxy = _actorProxyFactory.CreateActorProxy<ICartActor>(
        new ActorId(userId),
        "CartActor"
    );
    await proxy.AddItemAsync(item);
    return Ok();
}
```

## Configuring Actor Idle Timeout for Cart Expiry

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: cart-config
spec:
  entities:
  - type: CartActor
    actorIdleTimeout: 30m   # deactivate after 30 minutes of inactivity
    actorScanInterval: 10s
    drainOngoingCallTimeout: 5s
    drainRebalancedActors: true
```

## Deploying with a State Store

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis:6379"
  - name: actorStateStore
    value: "true"
```

## Summary

Dapr virtual actors provide a natural fit for per-user shopping cart state, offering automatic concurrency control, persistent state management, and idle timeout-based expiry. Each user maps to an actor ID, so there is no need for explicit locking or manual state cleanup. Combine this pattern with actor reminders to implement cart abandonment notifications without any additional scheduling infrastructure.
