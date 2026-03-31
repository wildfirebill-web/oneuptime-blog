# How to Install and Configure the Dapr .NET SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, .NET, SDK, C#, Configuration

Description: Install and configure the Dapr .NET SDK in ASP.NET Core applications, including NuGet packages, DaprClient setup, dependency injection, and middleware registration.

---

## Overview

The Dapr .NET SDK (`Dapr.Client` and `Dapr.AspNetCore`) provides a strongly-typed API for all Dapr building blocks. This guide covers installation, configuration, and the various patterns for using DaprClient in .NET applications.

## Step 1: Install NuGet Packages

```bash
# Core client package
dotnet add package Dapr.Client

# ASP.NET Core integration (controllers, middleware)
dotnet add package Dapr.AspNetCore

# Actors support
dotnet add package Dapr.Actors
dotnet add package Dapr.Actors.AspNetCore
```

## Step 2: Register Dapr in Program.cs

```csharp
using Dapr.Client;

var builder = WebApplication.CreateBuilder(args);

// Register DaprClient with dependency injection
builder.Services.AddDaprClient();

// Or with custom configuration
builder.Services.AddDaprClient(config =>
{
    config.UseDaprApiToken("your-api-token");
    config.UseHttpEndpoint("http://localhost:3500");
    config.UseGrpcEndpoint("http://localhost:50001");
});

// Register controllers with Dapr model binding
builder.Services.AddControllers().AddDapr();

var app = builder.Build();

// Add Dapr subscription middleware
app.UseCloudEvents();
app.MapControllers();
app.MapSubscribeHandler();

app.Run();
```

## Step 3: Use DaprClient via Dependency Injection

```csharp
[ApiController]
[Route("[controller]")]
public class OrdersController : ControllerBase
{
    private readonly DaprClient _daprClient;

    public OrdersController(DaprClient daprClient)
    {
        _daprClient = daprClient;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder(Order order)
    {
        // Save state
        await _daprClient.SaveStateAsync("statestore", order.Id, order);

        // Publish event
        await _daprClient.PublishEventAsync("pubsub", "order-created", order);

        return Ok(order);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(string id)
    {
        var order = await _daprClient.GetStateAsync<Order>("statestore", id);
        return order != null ? Ok(order) : NotFound();
    }
}
```

## Step 4: Use DaprClient Directly (Without DI)

```csharp
using Dapr.Client;

// Build client manually
using var client = new DaprClientBuilder()
    .UseJsonSerializationOptions(new System.Text.Json.JsonSerializerOptions
    {
        PropertyNameCaseInsensitive = true
    })
    .Build();

var result = await client.GetStateAsync<MyState>("statestore", "mykey");
```

## Step 5: Configure Timeout and Retry

```csharp
builder.Services.AddDaprClient(config =>
{
    config.UseGrpcEndpoint("http://localhost:50001");
    config.UseTimeout(TimeSpan.FromSeconds(30));
});
```

## Step 6: Environment Variable Configuration

The SDK respects standard Dapr environment variables:

```bash
export DAPR_HTTP_PORT=3500
export DAPR_GRPC_PORT=50001
export DAPR_API_TOKEN=my-token
```

```csharp
// SDK automatically reads these - no explicit config needed
builder.Services.AddDaprClient();
```

## Summary

The Dapr .NET SDK integrates cleanly with ASP.NET Core via `AddDaprClient()` and `AddDapr()` for controllers. DaprClient is registered as a singleton in the DI container, making it available throughout your application. The SDK supports both gRPC and HTTP transports, with all Dapr building blocks accessible through a unified, strongly-typed API.
