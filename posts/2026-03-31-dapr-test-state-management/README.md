# How to Test Dapr State Management Operations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, State Management, Testing, Unit Test, Integration Test

Description: Learn how to test Dapr State Management operations including save, get, delete, and transactions using mocks for unit tests and in-memory stores for integration tests.

---

## Testing State Operations

Dapr State Management operations - save, get, delete, bulk, and transaction - need testing at two levels: unit tests that mock the Dapr client and verify call arguments, and integration tests that use the in-memory state store to verify actual persistence behavior.

## Service Under Test

```csharp
// CartService.cs
public class CartService
{
    private readonly DaprClient _daprClient;
    private const string STORE = "statestore";

    public CartService(DaprClient daprClient)
    {
        _daprClient = daprClient;
    }

    public async Task<Cart> GetCartAsync(string userId)
    {
        return await _daprClient.GetStateAsync<Cart>(STORE, userId)
            ?? new Cart { UserId = userId, Items = new List<CartItem>() };
    }

    public async Task AddItemAsync(string userId, CartItem item)
    {
        var cart = await GetCartAsync(userId);
        var existing = cart.Items.FirstOrDefault(i => i.ProductId == item.ProductId);

        if (existing != null)
            existing.Quantity += item.Quantity;
        else
            cart.Items.Add(item);

        await _daprClient.SaveStateAsync(STORE, userId, cart);
    }

    public async Task CheckoutAsync(string userId)
    {
        // Atomic: delete cart + save checkout record
        var cart = await GetCartAsync(userId);
        var checkoutId = Guid.NewGuid().ToString();

        await _daprClient.ExecuteStateTransactionAsync(STORE, new List<StateTransactionRequest>
        {
            new StateTransactionRequest(userId, null, StateOperationType.Delete),
            new StateTransactionRequest(
                $"checkout:{checkoutId}",
                JsonSerializer.SerializeToUtf8Bytes(cart),
                StateOperationType.Upsert)
        });
    }
}
```

## Unit Tests

```csharp
// CartServiceTests.cs
public class CartServiceTests
{
    private readonly Mock<DaprClient> _mockDapr;
    private readonly CartService _service;

    public CartServiceTests()
    {
        _mockDapr = new Mock<DaprClient>();
        _service  = new CartService(_mockDapr.Object);
    }

    [Fact]
    public async Task GetCartAsync_ReturnsEmptyCartWhenNotFound()
    {
        _mockDapr
            .Setup(d => d.GetStateAsync<Cart>(
                "statestore", "user-1",
                It.IsAny<ConsistencyMode?>(),
                It.IsAny<IReadOnlyDictionary<string, string>>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync((Cart)null!);

        var cart = await _service.GetCartAsync("user-1");

        Assert.NotNull(cart);
        Assert.Empty(cart.Items);
        Assert.Equal("user-1", cart.UserId);
    }

    [Fact]
    public async Task AddItemAsync_SavesCartWithNewItem()
    {
        Cart? savedCart = null;

        _mockDapr
            .Setup(d => d.GetStateAsync<Cart>(
                "statestore", "user-1",
                It.IsAny<ConsistencyMode?>(),
                It.IsAny<IReadOnlyDictionary<string, string>>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(new Cart { UserId = "user-1", Items = new List<CartItem>() });

        _mockDapr
            .Setup(d => d.SaveStateAsync(
                "statestore", "user-1", It.IsAny<Cart>(),
                It.IsAny<StateOptions>(),
                It.IsAny<IReadOnlyDictionary<string, string>>(),
                It.IsAny<CancellationToken>()))
            .Callback<string, string, Cart, StateOptions,
                IReadOnlyDictionary<string, string>, CancellationToken>(
                (_, _, cart, _, _, _) => savedCart = cart)
            .Returns(Task.CompletedTask);

        await _service.AddItemAsync("user-1", new CartItem
        {
            ProductId = "sku-1",
            Quantity  = 3
        });

        Assert.NotNull(savedCart);
        Assert.Single(savedCart!.Items);
        Assert.Equal(3, savedCart.Items[0].Quantity);
    }
}
```

## Integration Test with In-Memory State Store

```csharp
// CartServiceIntegrationTests.cs
[Trait("Category", "Integration")]
public class CartServiceIntegrationTests
{
    // Requires: dapr run --app-id test-app --dapr-http-port 3500
    // With components using state.in-memory

    private readonly DaprClient _daprClient;
    private readonly CartService _service;

    public CartServiceIntegrationTests()
    {
        _daprClient = new DaprClientBuilder()
            .UseHttpEndpoint("http://localhost:3500")
            .Build();
        _service = new CartService(_daprClient);
    }

    [Fact]
    public async Task AddItem_PersistsInStateStore()
    {
        var userId = $"test-user-{Guid.NewGuid()}";

        await _service.AddItemAsync(userId, new CartItem
        {
            ProductId = "sku-test",
            Quantity  = 2
        });

        var cart = await _daprClient.GetStateAsync<Cart>("statestore", userId);
        Assert.NotNull(cart);
        Assert.Single(cart!.Items);
    }
}
```

## Summary

Test Dapr State Management at two levels. Unit tests mock `DaprClient` method calls and use `Callback` to capture saved objects for assertion. Integration tests use the in-memory Dapr state store and verify actual persistence through a real `DaprClient`. Covering both gives confidence in business logic correctness and correct Dapr API usage.
