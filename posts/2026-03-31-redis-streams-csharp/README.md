# How to Use Redis Streams in C#

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CSharp, Stream

Description: Learn how to produce and consume Redis Streams messages in C# using StackExchange.Redis with consumer groups and acknowledgment.

---

Redis Streams provide a persistent, ordered message log. Unlike Pub/Sub, messages are stored until explicitly deleted. StackExchange.Redis exposes Stream commands through `IDatabase`, allowing C# applications to build reliable event-driven systems.

## Produce Messages

```csharp
using StackExchange.Redis;

IDatabase db = redis.GetDatabase();

// Add a message to a stream
string id = await db.StreamAddAsync("orders", new NameValueEntry[]
{
    new NameValueEntry("orderId", "ord-001"),
    new NameValueEntry("amount", "49.99"),
    new NameValueEntry("status", "pending"),
});
Console.WriteLine($"Added message: {id}");
```

## Read All Messages

```csharp
StreamEntry[] messages = await db.StreamRangeAsync("orders", "-", "+");
foreach (StreamEntry msg in messages)
{
    Console.Write($"ID: {msg.Id}");
    foreach (NameValueEntry field in msg.Values)
    {
        Console.Write($" {field.Name}={field.Value}");
    }
    Console.WriteLine();
}
```

## Create a Consumer Group

```csharp
try
{
    await db.StreamCreateConsumerGroupAsync("orders", "processors", "$", true);
    // "$" = only new messages; "0" = from beginning
}
catch (RedisException ex) when (ex.Message.Contains("BUSYGROUP"))
{
    // Group already exists, this is fine
}
```

## Consumer Group - Read and Acknowledge

```csharp
public async Task ConsumeAsync(IDatabase db, string consumer, CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        StreamEntry[] entries = await db.StreamReadGroupAsync(
            "orders",
            "processors",
            consumer,
            ">",     // ">" = new messages not delivered to any consumer
            count: 10);

        if (entries.Length == 0)
        {
            await Task.Delay(1000, ct);
            continue;
        }

        foreach (StreamEntry entry in entries)
        {
            Console.WriteLine($"Processing: {entry.Id}");
            await ProcessOrder(entry);

            // Acknowledge after processing
            await db.StreamAcknowledgeAsync("orders", "processors", entry.Id);
        }
    }
}
```

## Process Pending Messages (Recovery)

```csharp
public async Task RecoverPendingAsync(IDatabase db, string consumer)
{
    StreamEntry[] pending = await db.StreamReadGroupAsync(
        "orders",
        "processors",
        consumer,
        "0",   // "0" = pending messages for this consumer
        count: 100);

    foreach (StreamEntry entry in pending)
    {
        Console.WriteLine($"Reprocessing: {entry.Id}");
        await ProcessOrder(entry);
        await db.StreamAcknowledgeAsync("orders", "processors", entry.Id);
    }
}
```

## Trim Stream Length

```csharp
// Keep only the last 1000 messages
await db.StreamTrimAsync("orders", 1000, useApproximateMaxLength: true);
```

## Stream Info

```csharp
StreamInfo info = await db.StreamInfoAsync("orders");
Console.WriteLine($"Length: {info.Length}");
Console.WriteLine($"First ID: {info.FirstEntry.Id}");
Console.WriteLine($"Last ID: {info.LastEntry.Id}");

StreamGroupInfo[] groups = await db.StreamGroupInfoAsync("orders");
foreach (StreamGroupInfo g in groups)
{
    Console.WriteLine($"Group: {g.Name}, Pending: {g.PendingMessageCount}");
}
```

## Background Service Consumer

```csharp
public class OrderConsumer : BackgroundService
{
    private readonly IConnectionMultiplexer _redis;

    public OrderConsumer(IConnectionMultiplexer redis) => _redis = redis;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        IDatabase db = _redis.GetDatabase();
        string consumer = $"worker-{Environment.MachineName}";
        await ConsumeAsync(db, consumer, stoppingToken);
    }
}
```

## Summary

Redis Streams in C# use `StreamAddAsync` to produce messages and `StreamReadGroupAsync` to consume them in consumer groups. Each message must be acknowledged with `StreamAcknowledgeAsync` after processing. Reading with ID `"0"` instead of `">"` recovers pending but unacknowledged messages after a crash. Wrapping the consumer loop in a `BackgroundService` integrates cleanly with ASP.NET Core's lifetime management.
