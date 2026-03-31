# How to Use Redis Lists in C#

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CSharp, List

Description: Learn how to use Redis lists in C# with StackExchange.Redis for push, pop, range queries, and building queues and activity feeds.

---

Redis lists are ordered sequences backed by a doubly-linked list. They support O(1) push and pop at both ends, making them useful for queues, stacks, activity feeds, and job buffers. StackExchange.Redis exposes list commands through `ListLeft*`, `ListRight*`, and `ListRange` methods.

## Push and Pop

```csharp
using StackExchange.Redis;

IDatabase db = redis.GetDatabase();

// LPUSH - push to left (head)
await db.ListLeftPushAsync("mylist", new RedisValue[] { "c", "b", "a" });
// list: [a, b, c]

// RPUSH - push to right (tail)
await db.ListRightPushAsync("mylist", new RedisValue[] { "d", "e" });
// list: [a, b, c, d, e]

// Pop from left
RedisValue left = await db.ListLeftPopAsync("mylist");
Console.WriteLine(left); // a

// Pop from right
RedisValue right = await db.ListRightPopAsync("mylist");
Console.WriteLine(right); // e
```

## Range and Index Access

```csharp
// Get all elements (0 to -1 = all)
RedisValue[] all = await db.ListRangeAsync("mylist", 0, -1);
Console.WriteLine(string.Join(", ", all)); // b, c, d

// Get element by index
RedisValue item = await db.ListGetByIndexAsync("mylist", 0);
Console.WriteLine(item); // b

// List length
long length = await db.ListLengthAsync("mylist");
Console.WriteLine(length); // 3
```

## Simple Queue (FIFO)

```csharp
public class RedisQueue(IDatabase db, string name)
{
    public Task EnqueueAsync(string item)
        => db.ListRightPushAsync(name, item);

    public async Task<string?> DequeueAsync()
    {
        RedisValue val = await db.ListLeftPopAsync(name);
        return val.IsNull ? null : val.ToString();
    }

    public Task<long> LengthAsync()
        => db.ListLengthAsync(name);
}

// Usage
var queue = new RedisQueue(db, "jobs:pending");
await queue.EnqueueAsync("job-001");
string? nextJob = await queue.DequeueAsync();
```

## Stack (LIFO)

```csharp
// Push to left, pop from left
await db.ListLeftPushAsync("stack", "first");
await db.ListLeftPushAsync("stack", "second");
await db.ListLeftPushAsync("stack", "third");

RedisValue top = await db.ListLeftPopAsync("stack");
Console.WriteLine(top); // third
```

## Update an Element

```csharp
// Set element at index 1
await db.ListSetByIndexAsync("mylist", 1, "new-value");
```

## Remove Elements

```csharp
// Remove 2 occurrences of "b" scanning from the head
long removed = await db.ListRemoveAsync("mylist", "b", 2);

// Trim to keep only indexes 0 through 99
await db.ListTrimAsync("recent:items", 0, 99);
```

## Activity Feed

```csharp
public class ActivityFeed(IDatabase db)
{
    private const int MaxItems = 100;

    public async Task RecordAsync(string userId, string event_)
    {
        string key = $"feed:{userId}";
        await db.ListLeftPushAsync(key, event_);
        await db.ListTrimAsync(key, 0, MaxItems - 1);
    }

    public async Task<IEnumerable<string>> GetRecentAsync(string userId, int count = 10)
    {
        RedisValue[] items = await db.ListRangeAsync($"feed:{userId}", 0, count - 1);
        return items.Select(v => v.ToString());
    }
}
```

## Move Between Lists Atomically

```csharp
// Move item from one list to another atomically
RedisValue moved = await db.ListMoveAsync(
    "queue:inbox",   // source
    "queue:processing", // destination
    ListSide.Left,   // pop from left of source
    ListSide.Right); // push to right of destination
```

## Summary

Redis lists in C# support push and pop at both ends with `ListLeftPushAsync`/`ListRightPushAsync` and corresponding pop methods. `ListRangeAsync` retrieves slices and `ListTrimAsync` caps list size to prevent unbounded growth. Use RPUSH + LPOP for a FIFO queue, LPUSH + LPOP for a stack, and `ListMoveAsync` for atomic handoff between processing queues.
