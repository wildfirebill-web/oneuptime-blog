# How to Use Actors for Shopping Cart State in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actors, Shopping Cart, State Management, Microservices

Description: Learn how to implement a shopping cart using Dapr virtual actors, providing per-user isolated state management with automatic concurrency control.

---

## Why Use Dapr Actors for Shopping Carts

A shopping cart is a natural fit for Dapr's virtual actor model. Each cart is owned by one user (actor instance), operations on it are naturally sequential (no concurrent modification conflicts), and the state needs to persist between sessions. Dapr handles actor placement, state storage, and concurrency automatically.

## Prerequisites

- Dapr CLI installed and initialized
- A state store configured (Redis or similar)
- A Dapr SDK for your language (.NET, Python, Java, Node.js)

## Define the Cart Interface (.NET)

```csharp
using Dapr.Actors;
using System.Collections.Generic;
using System.Threading.Tasks;

public interface IShoppingCartActor : IActor
{
    Task AddItemAsync(CartItem item);
    Task RemoveItemAsync(string productId);
    Task UpdateQuantityAsync(string productId, int quantity);
    Task<List<CartItem>> GetItemsAsync();
    Task<decimal> GetTotalAsync();
    Task ClearCartAsync();
    Task CheckoutAsync();
}

public record CartItem(string ProductId, string Name, decimal Price, int Quantity);
```

## Implement the Cart Actor (.NET)

```csharp
using Dapr.Actors.Runtime;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

[Actor(TypeName = "ShoppingCartActor")]
public class ShoppingCartActor : Actor, IShoppingCartActor
{
    private const string CartStateKey = "cart-items";

    public ShoppingCartActor(ActorHost host) : base(host) { }

    public async Task AddItemAsync(CartItem item)
    {
        var items = await GetItemsInternalAsync();
        var existing = items.FirstOrDefault(i => i.ProductId == item.ProductId);

        if (existing != null)
        {
            items.Remove(existing);
            items.Add(existing with { Quantity = existing.Quantity + item.Quantity });
        }
        else
        {
            items.Add(item);
        }

        await StateManager.SetStateAsync(CartStateKey, items);
        Console.WriteLine($"[Cart:{Id}] Added {item.Name} x{item.Quantity}");
    }

    public async Task RemoveItemAsync(string productId)
    {
        var items = await GetItemsInternalAsync();
        items.RemoveAll(i => i.ProductId == productId);
        await StateManager.SetStateAsync(CartStateKey, items);
        Console.WriteLine($"[Cart:{Id}] Removed product {productId}");
    }

    public async Task UpdateQuantityAsync(string productId, int quantity)
    {
        var items = await GetItemsInternalAsync();
        var existing = items.FirstOrDefault(i => i.ProductId == productId);
        if (existing == null) return;

        items.Remove(existing);
        if (quantity > 0)
            items.Add(existing with { Quantity = quantity });

        await StateManager.SetStateAsync(CartStateKey, items);
    }

    public async Task<List<CartItem>> GetItemsAsync()
    {
        return await GetItemsInternalAsync();
    }

    public async Task<decimal> GetTotalAsync()
    {
        var items = await GetItemsInternalAsync();
        return items.Sum(i => i.Price * i.Quantity);
    }

    public async Task ClearCartAsync()
    {
        await StateManager.SetStateAsync(CartStateKey, new List<CartItem>());
        Console.WriteLine($"[Cart:{Id}] Cart cleared");
    }

    public async Task CheckoutAsync()
    {
        var items = await GetItemsInternalAsync();
        var total = items.Sum(i => i.Price * i.Quantity);
        Console.WriteLine($"[Cart:{Id}] Checkout - {items.Count} items, total ${total:F2}");

        // Publish order event, clear cart, etc.
        await ClearCartAsync();
    }

    private async Task<List<CartItem>> GetItemsInternalAsync()
    {
        return await StateManager.GetOrAddStateAsync(CartStateKey, new List<CartItem>());
    }
}
```

## Implement the Cart Actor (Python)

```python
from dapr.actor import Actor
from typing import List, Optional
import json

class ShoppingCartActor(Actor):
    CART_KEY = "cart-items"

    async def add_item(self, item: dict):
        items = await self._get_items()
        existing = next((i for i in items if i["productId"] == item["productId"]), None)

        if existing:
            existing["quantity"] += item["quantity"]
        else:
            items.append(item)

        await self._state_manager.set_state(self.CART_KEY, items)
        print(f"[Cart:{self.id.id}] Added {item['name']} x{item['quantity']}")

    async def remove_item(self, product_id: str):
        items = await self._get_items()
        items = [i for i in items if i["productId"] != product_id]
        await self._state_manager.set_state(self.CART_KEY, items)

    async def get_items(self) -> list:
        return await self._get_items()

    async def get_total(self) -> float:
        items = await self._get_items()
        return sum(i["price"] * i["quantity"] for i in items)

    async def clear_cart(self):
        await self._state_manager.set_state(self.CART_KEY, [])

    async def _get_items(self) -> list:
        exists, items = await self._state_manager.try_get_state(self.CART_KEY)
        return items if exists and items else []
```

## Call the Cart Actor from a Web API

```csharp
// In an ASP.NET Controller or Minimal API
app.MapPost("/cart/{userId}/add", async (string userId, CartItem item, IActorProxyFactory factory) =>
{
    var actorId = new ActorId(userId);
    var cart = factory.CreateActorProxy<IShoppingCartActor>(actorId, "ShoppingCartActor");
    await cart.AddItemAsync(item);
    return Results.Ok();
});

app.MapGet("/cart/{userId}", async (string userId, IActorProxyFactory factory) =>
{
    var actorId = new ActorId(userId);
    var cart = factory.CreateActorProxy<IShoppingCartActor>(actorId, "ShoppingCartActor");
    var items = await cart.GetItemsAsync();
    var total = await cart.GetTotalAsync();
    return Results.Ok(new { items, total });
});

app.MapPost("/cart/{userId}/checkout", async (string userId, IActorProxyFactory factory) =>
{
    var actorId = new ActorId(userId);
    var cart = factory.CreateActorProxy<IShoppingCartActor>(actorId, "ShoppingCartActor");
    await cart.CheckoutAsync();
    return Results.Ok(new { message = "Checkout successful" });
});
```

## Benefits of the Actor Model for Shopping Carts

```text
Turn-based concurrency: All operations on a cart are serialized automatically.
No locking: You never write mutex or semaphore code.
Per-user isolation: Each user has their own actor instance with dedicated state.
Automatic persistence: State is saved to the configured store on every change.
Lazy activation: Actors are only activated when accessed, saving resources.
```

## Summary

Using Dapr virtual actors for shopping cart state provides automatic per-user isolation, built-in concurrency control through turn-based execution, and durable state persistence without custom locking code. Each user's cart is an actor instance that Dapr manages - activating it on demand, persisting its state, and ensuring operations are always serialized. This model scales naturally as the number of users grows across distributed deployments.
