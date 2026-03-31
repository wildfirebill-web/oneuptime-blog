# How to Test Dapr Pub/Sub Message Handlers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Testing, Message Handler, Test Automation

Description: Learn how to unit and integration test Dapr Pub/Sub message handler endpoints to verify subscription routing, message deserialization, and processing logic.

---

## What to Test in Pub/Sub Handlers

A Dapr pub/sub handler is an HTTP endpoint that receives a CloudEvents-wrapped message and returns 200 (ACK), 404 (drop), or 500 (NACK/retry). Tests should verify:

1. The `/dapr/subscribe` endpoint returns correct subscription definitions.
2. The handler deserializes the message correctly.
3. Business logic executes with the right parameters.
4. The correct HTTP status code is returned.

## Handler Under Test

```csharp
// OrderEventsController.cs
[ApiController]
public class OrderEventsController : ControllerBase
{
    private readonly IOrderProcessor _processor;
    private readonly ILogger<OrderEventsController> _logger;

    public OrderEventsController(
        IOrderProcessor processor,
        ILogger<OrderEventsController> logger)
    {
        _processor = processor;
        _logger = logger;
    }

    [HttpGet("/dapr/subscribe")]
    public IActionResult Subscribe()
    {
        return Ok(new[]
        {
            new
            {
                pubsubname = "pubsub",
                topic      = "order-created",
                route      = "/orders/created"
            }
        });
    }

    [HttpPost("/orders/created")]
    [Topic("pubsub", "order-created")]
    public async Task<IActionResult> HandleOrderCreated([FromBody] OrderCreatedEvent evt)
    {
        try
        {
            await _processor.ProcessNewOrderAsync(evt.OrderId, evt.CustomerId);
            return Ok();
        }
        catch (InvalidOrderException ex)
        {
            _logger.LogWarning("Dropping invalid order message: {Msg}", ex.Message);
            return NotFound(); // Drop the message
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Processing failed");
            return StatusCode(500); // Retry
        }
    }
}
```

## Unit Testing the Handler

```csharp
// OrderEventsControllerTests.cs
public class OrderEventsControllerTests
{
    private readonly Mock<IOrderProcessor> _mockProcessor;
    private readonly OrderEventsController _controller;

    public OrderEventsControllerTests()
    {
        _mockProcessor = new Mock<IOrderProcessor>();
        _controller = new OrderEventsController(
            _mockProcessor.Object,
            Mock.Of<ILogger<OrderEventsController>>());
    }

    [Fact]
    public void Subscribe_ReturnsCorrectSubscriptions()
    {
        var result = _controller.Subscribe() as OkObjectResult;
        var subs = result!.Value as IEnumerable<object>;

        Assert.NotNull(subs);
        Assert.Single(subs);
    }

    [Fact]
    public async Task HandleOrderCreated_ProcessesMessageAndReturns200()
    {
        _mockProcessor
            .Setup(p => p.ProcessNewOrderAsync("order-1", "cust-1"))
            .Returns(Task.CompletedTask);

        var evt = new OrderCreatedEvent { OrderId = "order-1", CustomerId = "cust-1" };
        var result = await _controller.HandleOrderCreated(evt);

        Assert.IsType<OkResult>(result);
        _mockProcessor.Verify(p => p.ProcessNewOrderAsync("order-1", "cust-1"), Times.Once);
    }

    [Fact]
    public async Task HandleOrderCreated_Returns404ForInvalidOrder()
    {
        _mockProcessor
            .Setup(p => p.ProcessNewOrderAsync(It.IsAny<string>(), It.IsAny<string>()))
            .ThrowsAsync(new InvalidOrderException("Order already cancelled"));

        var result = await _controller.HandleOrderCreated(
            new OrderCreatedEvent { OrderId = "bad-order", CustomerId = "cust-1" });

        Assert.IsType<NotFoundResult>(result);
    }

    [Fact]
    public async Task HandleOrderCreated_Returns500OnUnexpectedError()
    {
        _mockProcessor
            .Setup(p => p.ProcessNewOrderAsync(It.IsAny<string>(), It.IsAny<string>()))
            .ThrowsAsync(new Exception("Database connection lost"));

        var result = await _controller.HandleOrderCreated(
            new OrderCreatedEvent { OrderId = "order-2", CustomerId = "cust-2" });

        var statusResult = Assert.IsType<StatusCodeResult>(result);
        Assert.Equal(500, statusResult.StatusCode);
    }
}
```

## Integration Testing with Real Message Delivery

Use the Dapr HTTP API to publish a test message and verify your handler processes it:

```csharp
[Fact]
[Trait("Category", "Integration")]
public async Task PublishedMessage_IsProcessedByHandler()
{
    var daprClient = new DaprClientBuilder()
        .UseHttpEndpoint("http://localhost:3500")
        .Build();

    await daprClient.PublishEventAsync("pubsub", "order-created", new
    {
        orderId    = "integration-order-1",
        customerId = "integration-cust-1"
    });

    // Allow time for message delivery
    await Task.Delay(500);

    // Verify processing via state store
    var processed = await daprClient.GetStateAsync<bool>(
        "statestore", "processed-integration-order-1");

    Assert.True(processed);
}
```

## Summary

Dapr pub/sub handlers are plain HTTP controllers, making them easy to unit test without any Dapr infrastructure. Test the 200/404/500 status codes returned for different scenarios - 200 ACKs the message, 404 drops it, and 500 retries it. For integration tests, publish a message via the Dapr client and verify the side effect in your state store.
