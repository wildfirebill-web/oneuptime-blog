# How to Use Redis Sets and Sorted Sets in C#

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CSharp, Set

Description: Learn how to use Redis sets and sorted sets in C# with StackExchange.Redis for unique collections, set operations, and score-based ranking.

---

Redis sets hold unique unordered members. Sorted sets extend this with a score for each member, enabling ranked queries and range operations. Both are commonly used in C# applications for user tracking, tagging, and leaderboards.

## Sets - Basic Operations

```csharp
using StackExchange.Redis;

IDatabase db = redis.GetDatabase();

// Add members
await db.SetAddAsync("users:online", new RedisValue[] { "alice", "bob", "charlie" });
await db.SetAddAsync("users:premium", new RedisValue[] { "alice", "david" });

// Check membership
bool isOnline = await db.SetContainsAsync("users:online", "alice");
Console.WriteLine(isOnline); // true

// Count members
long count = await db.SetLengthAsync("users:online");
Console.WriteLine(count); // 3

// List all members
RedisValue[] members = await db.SetMembersAsync("users:online");
Console.WriteLine(string.Join(", ", members));
```

## Set Operations

```csharp
// Intersection
RedisValue[] both = await db.SetCombineAsync(SetOperation.Intersect,
    "users:online", "users:premium");
Console.WriteLine(string.Join(", ", both)); // alice

// Union
RedisValue[] all = await db.SetCombineAsync(SetOperation.Union,
    "users:online", "users:premium");

// Difference
RedisValue[] onlineNotPremium = await db.SetCombineAsync(SetOperation.Difference,
    "users:online", "users:premium");
Console.WriteLine(string.Join(", ", onlineNotPremium)); // bob, charlie
```

## Remove and Pop

```csharp
await db.SetRemoveAsync("users:online", "bob");

// Pop a random member
RedisValue popped = await db.SetPopAsync("users:online");
Console.WriteLine(popped);

// Get random members without removing
RedisValue[] randoms = await db.SetRandomMembersAsync("users:online", 2);
```

## Sorted Sets - Basic Operations

```csharp
// Add members with scores
await db.SortedSetAddAsync("leaderboard", new SortedSetEntry[]
{
    new SortedSetEntry("alice", 1500),
    new SortedSetEntry("bob", 2300),
    new SortedSetEntry("charlie", 900),
});

// Get rank (0-indexed, lowest score first)
long? rank = await db.SortedSetRankAsync("leaderboard", "alice");
Console.WriteLine(rank); // 1

// Get rank by highest score
long? revRank = await db.SortedSetRankAsync("leaderboard", "bob", Order.Descending);
Console.WriteLine(revRank); // 0

// Get score
double? score = await db.SortedSetScoreAsync("leaderboard", "alice");
Console.WriteLine(score); // 1500
```

## Range Queries

```csharp
// Top 3 by highest score
SortedSetEntry[] top3 = await db.SortedSetRangeByRankWithScoresAsync(
    "leaderboard", 0, 2, Order.Descending);

foreach (SortedSetEntry entry in top3)
{
    Console.WriteLine($"{entry.Element}: {entry.Score}");
}

// Members with score between 1000 and 2000
RedisValue[] inRange = await db.SortedSetRangeByScoreAsync(
    "leaderboard", 1000, 2000);
Console.WriteLine(string.Join(", ", inRange)); // alice
```

## Increment Score

```csharp
double newScore = await db.SortedSetIncrementAsync("leaderboard", "alice", 100);
Console.WriteLine(newScore); // 1600
```

## Remove Members

```csharp
await db.SortedSetRemoveAsync("leaderboard", "charlie");

// Remove lowest 2 scores
await db.SortedSetRemoveRangeByRankAsync("leaderboard", 0, 1);

// Remove by score range
await db.SortedSetRemoveRangeByScoreAsync("leaderboard", double.NegativeInfinity, 1000);
```

## Summary

Redis sets in C# handle unique member collections with fast membership tests and set algebra through `SetCombineAsync`. Sorted sets extend this with per-member scores for ranked queries using `SortedSetRangeByRankWithScoresAsync`. Both types are commonly used in C# web applications for active user tracking, feature flags, and leaderboards.
