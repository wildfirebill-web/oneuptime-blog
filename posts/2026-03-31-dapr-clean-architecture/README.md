# How to Use Dapr with Clean Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Clean Architecture, Design Pattern, Microservice, Architecture

Description: Structure Dapr microservices using Clean Architecture to keep business logic independent of Dapr building blocks through interfaces and dependency injection.

---

## Clean Architecture and Dapr

Clean Architecture organizes code into concentric layers: Entities (core domain), Use Cases (application logic), Interface Adapters, and Frameworks/Drivers. Dapr components (state stores, pub/sub, service invocation) belong in the outermost layer - they should never leak into your domain or use case layers.

## Project Structure

```
src/
- Domain/
  - Entities/
    - Order.cs
  - Interfaces/
    - IOrderRepository.cs
    - IEventPublisher.cs
- Application/
  - UseCases/
    - CreateOrderUseCase.cs
- Infrastructure/
  - Dapr/
    - DaprOrderRepository.cs    # Implements IOrderRepository using Dapr state
    - DaprEventPublisher.cs     # Implements IEventPublisher using Dapr pub/sub
- Presentation/
  - Controllers/
    - OrderController.cs
```

## Define Domain Interfaces

Your domain layer knows nothing about Dapr:

```csharp
// Domain/Interfaces/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order> GetByIdAsync(string orderId);
    Task SaveAsync(Order order);
}

// Domain/Interfaces/IEventPublisher.cs
public interface IEventPublisher
{
    Task PublishAsync<T>(string topic, T @event);
}
```

## Implement Dapr Adapters in Infrastructure Layer

```csharp
// Infrastructure/Dapr/DaprOrderRepository.cs
using Dapr.Client;

public class DaprOrderRepository : IOrderRepository
{
    private readonly DaprClient _dapr;
    private const string StoreName = "statestore";

    public DaprOrderRepository(DaprClient dapr)
    {
        _dapr = dapr;
    }

    public async Task<Order> GetByIdAsync(string orderId)
    {
        return await _dapr.GetStateAsync<Order>(StoreName, orderId);
    }

    public async Task SaveAsync(Order order)
    {
        await _dapr.SaveStateAsync(StoreName, order.Id, order);
    }
}

// Infrastructure/Dapr/DaprEventPublisher.cs
public class DaprEventPublisher : IEventPublisher
{
    private readonly DaprClient _dapr;
    private const string PubSubName = "pubsub";

    public DaprEventPublisher(DaprClient dapr) => _dapr = dapr;

    public async Task PublishAsync<T>(string topic, T @event)
    {
        await _dapr.PublishEventAsync(PubSubName, topic, @event);
    }
}
```

## Use Case - No Dapr Dependency

```csharp
// Application/UseCases/CreateOrderUseCase.cs
public class CreateOrderUseCase
{
    private readonly IOrderRepository _repository;
    private readonly IEventPublisher _publisher;

    public CreateOrderUseCase(IOrderRepository repository, IEventPublisher publisher)
    {
        _repository = repository;
        _publisher = publisher;
    }

    public async Task<Order> ExecuteAsync(CreateOrderRequest request)
    {
        var order = new Order(request.CustomerId, request.Items);
        await _repository.SaveAsync(order);
        await _publisher.PublishAsync("order-created", new OrderCreatedEvent(order.Id));
        return order;
    }
}
```

## Register Dependencies in Program.cs

```csharp
// Presentation/Program.cs
builder.Services.AddDaprClient();
builder.Services.AddScoped<IOrderRepository, DaprOrderRepository>();
builder.Services.AddScoped<IEventPublisher, DaprEventPublisher>();
builder.Services.AddScoped<CreateOrderUseCase>();
```

## Unit Testing Without Dapr

Because the use case depends on interfaces, you can unit test it without a running Dapr sidecar:

```csharp
[Fact]
public async Task CreateOrder_SavesAndPublishes()
{
    var mockRepo = new Mock<IOrderRepository>();
    var mockPublisher = new Mock<IEventPublisher>();

    var useCase = new CreateOrderUseCase(mockRepo.Object, mockPublisher.Object);
    var result = await useCase.ExecuteAsync(new CreateOrderRequest("cust-1", new[] { "item-1" }));

    mockRepo.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);
    mockPublisher.Verify(p => p.PublishAsync("order-created", It.IsAny<OrderCreatedEvent>()), Times.Once);
}
```

## Summary

Using Dapr with Clean Architecture keeps business logic independent of Dapr building blocks by placing Dapr adapters in the infrastructure layer behind domain interfaces. This enables full unit test coverage of use cases without a running Dapr sidecar and makes it easy to swap Dapr components for different implementations without touching domain code.
