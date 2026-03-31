# How to Use Dapr Workflow with .NET SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, .NET, Workflow, C#, Orchestration

Description: Build reliable long-running workflows in .NET using the Dapr Workflow SDK, including workflow definitions, activities, child workflows, and retry policies.

---

## Overview

Dapr Workflow (built on Durable Task Framework) enables long-running, fault-tolerant process orchestration in .NET. Workflows persist their state so they survive restarts and failures, making them ideal for order processing, approval flows, and saga patterns.

## Setup

```bash
dotnet add package Dapr.Workflow
```

```csharp
// Program.cs
builder.Services.AddDaprWorkflow(options =>
{
    options.RegisterWorkflow<OrderFulfillmentWorkflow>();
    options.RegisterActivity<ValidateOrderActivity>();
    options.RegisterActivity<ProcessPaymentActivity>();
    options.RegisterActivity<ShipOrderActivity>();
    options.RegisterActivity<SendNotificationActivity>();
});
```

## Define Activities

```csharp
public class ProcessPaymentActivity : WorkflowActivity<PaymentRequest, PaymentResult>
{
    private readonly PaymentService _paymentService;

    public ProcessPaymentActivity(PaymentService paymentService)
        => _paymentService = paymentService;

    public override async Task<PaymentResult> RunAsync(
        WorkflowActivityContext context,
        PaymentRequest input)
    {
        Console.WriteLine($"Processing payment for order {input.OrderId}");
        return await _paymentService.ChargeAsync(input.CustomerId, input.Amount);
    }
}

public class ShipOrderActivity : WorkflowActivity<ShipRequest, ShipResult>
{
    public override async Task<ShipResult> RunAsync(
        WorkflowActivityContext context,
        ShipRequest input)
    {
        // Call shipping service
        return new ShipResult { TrackingNumber = Guid.NewGuid().ToString() };
    }
}
```

## Define the Workflow

```csharp
public class OrderFulfillmentWorkflow : Workflow<OrderRequest, OrderResult>
{
    public override async Task<OrderResult> RunAsync(
        WorkflowContext context,
        OrderRequest input)
    {
        // Step 1: Validate order
        var validation = await context.CallActivityAsync<ValidationResult>(
            nameof(ValidateOrderActivity), input);

        if (!validation.IsValid)
            return new OrderResult { Status = "rejected", Reason = validation.Error };

        // Step 2: Process payment (with retry)
        var retryPolicy = new WorkflowTaskOptions
        {
            RetryPolicy = new WorkflowRetryPolicy(
                maxNumberOfAttempts: 3,
                firstRetryInterval: TimeSpan.FromSeconds(5))
        };

        var payment = await context.CallActivityAsync<PaymentResult>(
            nameof(ProcessPaymentActivity),
            new PaymentRequest { OrderId = input.OrderId, CustomerId = input.CustomerId, Amount = input.Total },
            retryPolicy);

        if (!payment.Success)
            return new OrderResult { Status = "payment_failed" };

        // Step 3: Ship order
        var ship = await context.CallActivityAsync<ShipResult>(
            nameof(ShipOrderActivity),
            new ShipRequest { OrderId = input.OrderId, Address = input.ShippingAddress });

        // Step 4: Notify customer
        await context.CallActivityAsync(
            nameof(SendNotificationActivity),
            new NotificationRequest { Email = input.CustomerEmail, TrackingNumber = ship.TrackingNumber });

        return new OrderResult { Status = "fulfilled", TrackingNumber = ship.TrackingNumber };
    }
}
```

## Start and Monitor a Workflow

```csharp
[ApiController]
[Route("[controller]")]
public class OrdersController : ControllerBase
{
    private readonly DaprWorkflowClient _workflowClient;

    public OrdersController(DaprWorkflowClient workflowClient)
        => _workflowClient = workflowClient;

    [HttpPost]
    public async Task<IActionResult> CreateOrder(OrderRequest request)
    {
        var instanceId = $"order-{Guid.NewGuid():N}";

        await _workflowClient.ScheduleNewWorkflowAsync(
            name: nameof(OrderFulfillmentWorkflow),
            instanceId: instanceId,
            input: request);

        return Accepted(new { workflowId = instanceId });
    }

    [HttpGet("{id}/status")]
    public async Task<IActionResult> GetStatus(string id)
    {
        var state = await _workflowClient.GetWorkflowStateAsync(id);
        return Ok(new { status = state.RuntimeStatus.ToString(), output = state.ReadOutputAs<OrderResult>() });
    }
}
```

## Summary

Dapr Workflow in .NET provides durable, fault-tolerant orchestration built on the Durable Task Framework. Activities are reusable units of work, while workflow definitions coordinate them with retry policies and conditional logic. Workflow state is persisted automatically, so long-running processes like order fulfillment survive application restarts without losing progress.
