# How to Test Dapr .NET Applications with Testcontainers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Dotnet, Testing, Testcontainer, Integration Test, Microservice

Description: Use Testcontainers for .NET to spin up real Dapr sidecars and dependencies in integration tests, ensuring your Dapr-powered services behave correctly end-to-end.

---

## Overview

Integration testing Dapr applications requires a running Dapr sidecar alongside your service. Testcontainers for .NET makes this reproducible by spinning up Docker containers for each test run, giving you a real Dapr environment without relying on a shared cluster.

## Installing Packages

```bash
dotnet add package Testcontainers
dotnet add package Testcontainers.Dapr
dotnet add package Microsoft.AspNetCore.Mvc.Testing
dotnet add package xunit
```

## Setting Up the Test Fixture

Create a fixture that starts a Dapr sidecar container alongside your ASP.NET Core test server:

```csharp
using Testcontainers.Dapr;
using Microsoft.AspNetCore.Mvc.Testing;

public class DaprFixture : IAsyncLifetime
{
    private DaprContainer _daprContainer = null!;
    public HttpClient Client { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        _daprContainer = new DaprBuilder()
            .WithAppId("order-service")
            .WithAppPort(5001)
            .WithComponentsPath("./components")
            .Build();

        await _daprContainer.StartAsync();

        var factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(host =>
            {
                host.UseSetting("DAPR_HTTP_PORT", _daprContainer.DaprHttpPort.ToString());
            });

        Client = factory.CreateClient();
    }

    public async Task DisposeAsync()
    {
        await _daprContainer.DisposeAsync();
    }
}
```

## Writing an Integration Test

```csharp
public class OrderServiceTests : IClassFixture<DaprFixture>
{
    private readonly HttpClient _client;

    public OrderServiceTests(DaprFixture fixture)
    {
        _client = fixture.Client;
    }

    [Fact]
    public async Task PostOrder_ShouldReturnCreated()
    {
        var order = new { Id = "ord-1", Product = "Widget", Quantity = 2 };
        var response = await _client.PostAsJsonAsync("/orders", order);

        response.EnsureSuccessStatusCode();
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }

    [Fact]
    public async Task GetOrder_AfterPost_ShouldReturnOrder()
    {
        var order = new { Id = "ord-2", Product = "Gadget", Quantity = 1 };
        await _client.PostAsJsonAsync("/orders", order);

        var result = await _client.GetFromJsonAsync<dynamic>("/orders/ord-2");
        Assert.NotNull(result);
    }
}
```

## Using In-Memory State Store for Tests

Create a test-specific Dapr component pointing to an in-memory store so tests are isolated:

```yaml
# components/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.in-memory
  version: v1
```

## Running Tests

```bash
dotnet test --filter Category=Integration
```

## Summary

Testcontainers for .NET combined with the Dapr container module enables reliable, isolated integration tests that run against a real Dapr sidecar. Using an in-memory state store component for testing keeps tests fast and deterministic, while the full Dapr sidecar ensures pub/sub routing and service invocation behave exactly as they would in production.
