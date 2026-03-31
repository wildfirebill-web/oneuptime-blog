# How to Use Redis Pub/Sub in C#

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CSharp, Pub/Sub

Description: Learn how to publish messages and subscribe to Redis channels in C# using StackExchange.Redis with proper async handling and pattern subscriptions.

---

Redis Pub/Sub lets services broadcast messages to subscribers on named channels. StackExchange.Redis provides `ISubscriber` for publishing and subscribing, with both callback-based and channel-reader APIs.

## Basic Publish and Subscribe

```csharp
using StackExchange.Redis;

ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("localhost:6379");
ISubscriber sub = redis.GetSubscriber();

// Subscribe
await sub.SubscribeAsync(RedisChannel.Literal("notifications"), (channel, message) =>
{
    Console.WriteLine($"[{channel}] {message}");
});

// Publish from the same or a different connection
IDatabase db = redis.GetDatabase();
await db.PublishAsync(RedisChannel.Literal("notifications"), "Hello, subscribers!");
```

## Channel Reader API (Recommended)

The channel reader API is cleaner for async processing:

```csharp
ChannelMessageQueue queue = await sub.SubscribeAsync(
    RedisChannel.Literal("events"));

// Process messages in a loop
await foreach (ChannelMessage msg in queue)
{
    Console.WriteLine($"Received: {msg.Message}");
}
```

## Pattern Subscription

```csharp
// Subscribe to all channels matching "user:*:updated"
await sub.SubscribeAsync(RedisChannel.Pattern("user:*:updated"), (channel, message) =>
{
    Console.WriteLine($"Pattern match on {channel}: {message}");
});
```

## Background Subscriber Service (ASP.NET Core)

```csharp
public class NotificationSubscriber : BackgroundService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<NotificationSubscriber> _logger;

    public NotificationSubscriber(IConnectionMultiplexer redis,
        ILogger<NotificationSubscriber> logger)
    {
        _redis = redis;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        ISubscriber sub = _redis.GetSubscriber();
        ChannelMessageQueue queue = await sub.SubscribeAsync(
            RedisChannel.Literal("alerts"));

        await foreach (ChannelMessage msg in queue.WithCancellation(stoppingToken))
        {
            _logger.LogInformation("Alert received: {Message}", msg.Message);
            await ProcessAlert(msg.Message!);
        }
    }

    private Task ProcessAlert(string message)
    {
        // Handle the alert
        return Task.CompletedTask;
    }
}
```

Register it:

```csharp
builder.Services.AddHostedService<NotificationSubscriber>();
```

## Publisher Service

```csharp
public class EventPublisher(IConnectionMultiplexer redis)
{
    private readonly ISubscriber _sub = redis.GetSubscriber();

    public async Task PublishAsync(string channel, string message)
    {
        long recipients = await _sub.PublishAsync(
            RedisChannel.Literal(channel), message);
        Console.WriteLine($"Delivered to {recipients} subscribers");
    }
}
```

## Unsubscribe

```csharp
// Unsubscribe from a specific channel
await sub.UnsubscribeAsync(RedisChannel.Literal("notifications"));

// Unsubscribe from all
await sub.UnsubscribeAllAsync();
```

## Pub/Sub vs Streams

- Pub/Sub: messages are not persisted and are lost if no subscriber is active
- Redis Streams: messages persist and support consumer groups for reliable delivery

Use Pub/Sub for live notifications where missing a message is acceptable. Use Streams when every message must be processed.

## Summary

StackExchange.Redis Pub/Sub uses `ISubscriber` from `redis.GetSubscriber()`. The `ChannelMessageQueue` API with `await foreach` is the cleanest way to process messages. Pattern subscriptions match multiple channels with a glob pattern. In ASP.NET Core, wrap the subscriber loop in a `BackgroundService` so it runs for the lifetime of the application and respects cancellation on shutdown.
