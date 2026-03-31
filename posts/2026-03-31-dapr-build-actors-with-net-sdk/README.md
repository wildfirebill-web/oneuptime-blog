# How to Build Dapr Actors with .NET SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actors, .NET, C#, Virtual Actors

Description: Learn how to implement Dapr virtual actors using the .NET SDK, covering actor interfaces, state management, timers, reminders, and deployment patterns.

---

## What Are Dapr Virtual Actors

Dapr implements the virtual actor pattern, where actors are automatically activated on demand and garbage-collected when idle. Each actor has a unique ID, isolated state, and turn-based concurrency - only one operation runs on an actor at a time. With the .NET SDK, you define actors as C# classes backed by an interface.

## Project Setup

```bash
dotnet new webapi -n ActorDemo
cd ActorDemo
dotnet add package Dapr.Actors.AspNetCore
```

## Defining an Actor Interface

The interface defines the contract for actor operations. It must extend `IActor`:

```csharp
using Dapr.Actors;

public interface ICounterActor : IActor
{
    Task IncrementAsync(int amount);
    Task<int> GetCountAsync();
    Task ResetAsync();
}
```

## Implementing the Actor

Implement the interface by extending `Actor` and implementing your interface:

```csharp
using Dapr.Actors;
using Dapr.Actors.Runtime;

[Actor(TypeName = "CounterActor")]
public class CounterActor : Actor, ICounterActor
{
    private const string CountStateKey = "count";

    public CounterActor(ActorHost host) : base(host)
    {
    }

    public async Task IncrementAsync(int amount)
    {
        int current = await StateManager.GetOrAddStateAsync(CountStateKey, 0);
        int newValue = current + amount;
        await StateManager.SetStateAsync(CountStateKey, newValue);
        Logger.LogInformation("Actor {Id}: incremented by {Amount}, now {Value}",
            Id, amount, newValue);
    }

    public async Task<int> GetCountAsync()
    {
        return await StateManager.GetOrAddStateAsync(CountStateKey, 0);
    }

    public async Task ResetAsync()
    {
        await StateManager.SetStateAsync(CountStateKey, 0);
        Logger.LogInformation("Actor {Id}: reset to 0", Id);
    }
}
```

## Registering Actors

Register actors in your ASP.NET Core application:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddActors(options =>
{
    options.Actors.RegisterActor<CounterActor>();
    options.ActorIdleTimeout = TimeSpan.FromMinutes(5);
    options.DrainOngoingCallTimeout = TimeSpan.FromSeconds(30);
});

var app = builder.Build();
app.MapActorsHandlers(); // Required - registers Dapr actor endpoints
app.Run();
```

## Invoking Actors from Client Code

Use `ActorProxy` to create a client proxy for any actor:

```csharp
using Dapr.Actors;
using Dapr.Actors.Client;

[ApiController]
[Route("api/counters")]
public class CounterController : ControllerBase
{
    [HttpPost("{id}/increment")]
    public async Task<IActionResult> Increment(string id, [FromQuery] int amount = 1)
    {
        var actorId = new ActorId(id);
        var proxy = ActorProxy.Create<ICounterActor>(actorId, "CounterActor");
        await proxy.IncrementAsync(amount);
        return Ok(new { message = $"Counter {id} incremented by {amount}" });
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetCount(string id)
    {
        var actorId = new ActorId(id);
        var proxy = ActorProxy.Create<ICounterActor>(actorId, "CounterActor");
        int count = await proxy.GetCountAsync();
        return Ok(new { id, count });
    }

    [HttpPost("{id}/reset")]
    public async Task<IActionResult> Reset(string id)
    {
        var actorId = new ActorId(id);
        var proxy = ActorProxy.Create<ICounterActor>(actorId, "CounterActor");
        await proxy.ResetAsync();
        return Ok(new { message = $"Counter {id} reset" });
    }
}
```

## Adding Actor Timers

Timers run periodically but are not persisted across actor deactivation. Use them for in-memory periodic tasks:

```csharp
[Actor(TypeName = "HeartbeatActor")]
public class HeartbeatActor : Actor, IHeartbeatActor
{
    protected override async Task OnActivateAsync()
    {
        await RegisterTimerAsync(
            "heartbeat-timer",
            nameof(SendHeartbeatAsync),
            null,
            dueTime: TimeSpan.Zero,
            period: TimeSpan.FromSeconds(10));
    }

    public Task SendHeartbeatAsync(byte[] data)
    {
        Logger.LogInformation("Heartbeat from actor {Id} at {Time}", Id, DateTime.UtcNow);
        return Task.CompletedTask;
    }
}
```

## Adding Actor Reminders

Reminders persist across actor deactivation and reactivation, making them reliable for long-term scheduling:

```csharp
[Actor(TypeName = "ReminderActor")]
public class ReminderActor : Actor, IReminderActor, IRemindable
{
    public async Task ScheduleReminderAsync(string name, TimeSpan dueTime)
    {
        await RegisterReminderAsync(
            name,
            null,
            dueTime,
            period: TimeSpan.FromDays(1));
        Logger.LogInformation("Reminder {Name} scheduled for actor {Id}", name, Id);
    }

    public async Task ReceiveReminderAsync(string reminderName, byte[] state,
        TimeSpan dueTime, TimeSpan period)
    {
        Logger.LogInformation("Reminder {Name} fired for actor {Id}", reminderName, Id);
        await ProcessReminderAsync(reminderName);
    }
}
```

## Running the Application

```bash
dapr run --app-id counter-service \
         --app-port 5000 \
         --dapr-http-port 3500 \
         --resources-path ./components \
         -- dotnet run
```

Test the actor:

```bash
# Increment counter for user-42
curl -X POST http://localhost:5000/api/counters/user-42/increment?amount=5

# Get current value
curl http://localhost:5000/api/counters/user-42
```

## Summary

Dapr virtual actors in .NET provide a clean, strongly-typed programming model for managing distributed state. Define your actor contract as an interface, implement it by extending the `Actor` base class, and register it with the ASP.NET Core pipeline. Timers handle periodic in-process work, while reminders provide durable scheduling that survives restarts, making the .NET actor SDK well-suited for stateful, long-lived entities in distributed systems.
