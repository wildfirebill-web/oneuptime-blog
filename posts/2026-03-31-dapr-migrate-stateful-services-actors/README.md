# How to Migrate Stateful Services to Dapr Actors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Stateful Service, Migration, Virtual Actor

Description: Learn how to migrate stateful microservices with manual state management to Dapr Actors for automatic state persistence and turn-based concurrency.

---

## The Problem with Manual Stateful Services

A stateful service that holds user session data or device state in memory breaks when scaled beyond one replica. You need sticky sessions, external state stores, and custom locking. Dapr Actors solve this with the Virtual Actor pattern: each actor is identified by an ID, state is automatically persisted, and turn-based concurrency prevents race conditions.

## Before: Manual Stateful Service

```csharp
// DeviceStateService.cs - manual state management
public class DeviceStateService
{
    private readonly IDatabase _db;
    private readonly IMemoryCache _cache;
    private readonly SemaphoreSlim _lock = new SemaphoreSlim(1, 1);

    public async Task<DeviceState> GetStateAsync(string deviceId)
    {
        if (_cache.TryGetValue(deviceId, out DeviceState cached))
            return cached;

        var state = await _db.GetAsync<DeviceState>(deviceId);
        _cache.Set(deviceId, state, TimeSpan.FromMinutes(5));
        return state;
    }

    public async Task UpdateStateAsync(string deviceId, DeviceCommand command)
    {
        await _lock.WaitAsync();
        try
        {
            var state = await GetStateAsync(deviceId);
            state.Apply(command);
            await _db.SaveAsync(deviceId, state);
            _cache.Set(deviceId, state, TimeSpan.FromMinutes(5));
        }
        finally
        {
            _lock.Release();
        }
    }
}
```

## After: Dapr Actor

Define the actor interface:

```csharp
// IDeviceActor.cs
public interface IDeviceActor : IActor
{
    Task<DeviceState> GetStateAsync();
    Task ApplyCommandAsync(DeviceCommand command);
    Task ResetAsync();
}
```

Implement the actor:

```csharp
// DeviceActor.cs
[Actor(TypeName = "DeviceActor")]
public class DeviceActor : Actor, IDeviceActor
{
    private const string StateKey = "device-state";

    public DeviceActor(ActorHost host) : base(host) { }

    public async Task<DeviceState> GetStateAsync()
    {
        var state = await StateManager.TryGetStateAsync<DeviceState>(StateKey);
        return state.HasValue ? state.Value : new DeviceState();
    }

    public async Task ApplyCommandAsync(DeviceCommand command)
    {
        var state = await GetStateAsync();
        state.Apply(command);
        await StateManager.SetStateAsync(StateKey, state);
        await StateManager.SaveStateAsync();
    }

    public async Task ResetAsync()
    {
        await StateManager.RemoveStateAsync(StateKey);
    }
}
```

## Calling the Actor from a Service

```csharp
// DeviceController.cs
[ApiController]
[Route("devices")]
public class DeviceController : ControllerBase
{
    private readonly IActorProxyFactory _actorProxyFactory;

    public DeviceController(IActorProxyFactory actorProxyFactory)
    {
        _actorProxyFactory = actorProxyFactory;
    }

    [HttpGet("{deviceId}/state")]
    public async Task<IActionResult> GetState(string deviceId)
    {
        var actor = _actorProxyFactory.CreateActorProxy<IDeviceActor>(
            new ActorId(deviceId),
            "DeviceActor");

        var state = await actor.GetStateAsync();
        return Ok(state);
    }

    [HttpPost("{deviceId}/command")]
    public async Task<IActionResult> ApplyCommand(
        string deviceId, [FromBody] DeviceCommand command)
    {
        var actor = _actorProxyFactory.CreateActorProxy<IDeviceActor>(
            new ActorId(deviceId),
            "DeviceActor");

        await actor.ApplyCommandAsync(command);
        return Ok();
    }
}
```

## Registration

```csharp
// Program.cs
builder.Services.AddActors(options =>
{
    options.Actors.RegisterActor<DeviceActor>();
    options.ActorIdleTimeout = TimeSpan.FromMinutes(30);
});

// Map actor handlers
app.MapActorsHandlers();
```

## Summary

Migrating stateful services to Dapr Actors removes manual locking, caching, and persistence code. Each actor instance is identified by a string ID (the device ID, user ID, etc.), its state is automatically saved to the configured state store, and turn-based concurrency prevents race conditions without `SemaphoreSlim`. Actors idle out automatically when not in use, freeing resources.
