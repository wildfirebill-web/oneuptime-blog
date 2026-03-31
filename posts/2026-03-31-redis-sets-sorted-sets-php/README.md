# How to Use Redis Sets and Sorted Sets in PHP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, PHP, Set, Sorted Set, Leaderboard

Description: Learn how to use Redis Sets and Sorted Sets in PHP with phpredis to build unique collections, tag systems, and score-based leaderboards.

---

Redis Sets store unique unordered members, while Sorted Sets add a score to each member for ordered retrieval. Both are powerful for tag systems, friend lists, and leaderboards.

## Redis Sets with phpredis

```php
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

// Add members
$redis->sAdd('tags:post:42', 'php', 'redis', 'tutorial');

// Get all members
$tags = $redis->sMembers('tags:post:42');
print_r($tags); // ['php', 'redis', 'tutorial'] (unordered)

// Check membership
$isMember = $redis->sIsMember('tags:post:42', 'redis');
echo $isMember ? "yes" : "no"; // yes

// Count members
$count = $redis->sCard('tags:post:42');
echo "Tag count: $count\n"; // 3

// Remove a member
$redis->sRem('tags:post:42', 'tutorial');
```

## Set Operations

```php
// Union of two sets
$redis->sAdd('tags:post:1', 'php', 'redis');
$redis->sAdd('tags:post:2', 'redis', 'mysql', 'php');

$union = $redis->sUnion('tags:post:1', 'tags:post:2');
// ['php', 'redis', 'mysql']

$intersection = $redis->sInter('tags:post:1', 'tags:post:2');
// ['php', 'redis']

$difference = $redis->sDiff('tags:post:1', 'tags:post:2');
// [] (post:1 has nothing post:2 does not)
```

## Redis Sorted Sets

```php
// Add members with scores
$redis->zAdd('leaderboard', 1500, 'alice');
$redis->zAdd('leaderboard', 2300, 'bob');
$redis->zAdd('leaderboard', 1900, 'carol');
$redis->zAdd('leaderboard', 3100, 'dave');

// Get top 3 (highest scores first)
$top3 = $redis->zRevRange('leaderboard', 0, 2, true);
// ['dave' => 3100, 'bob' => 2300, 'carol' => 1900]

// Get rank (0-based, ascending order)
$rank = $redis->zRank('leaderboard', 'alice');
echo "Alice rank: " . ($rank + 1); // 1 (lowest)

// Reverse rank (highest score = rank 0)
$topRank = $redis->zRevRank('leaderboard', 'dave');
echo "Dave top rank: " . ($topRank + 1); // 1
```

## Score-Based Range Queries

```php
// Members with scores between 1500 and 2500
$mid = $redis->zRangeByScore('leaderboard', 1500, 2500);
// ['alice', 'carol', 'bob']

// With scores included
$mid = $redis->zRangeByScore('leaderboard', 1500, 2500, ['withscores' => true]);
print_r($mid);
```

## Incrementing Scores

```php
// Increment alice's score (for example, a game event)
$newScore = $redis->zIncrBy('leaderboard', 200, 'alice');
echo "Alice new score: $newScore"; // 1700
```

## Expiry-Based Sorted Set (Priority Queue)

```php
function scheduleJob(Redis $redis, string $jobId, int $runAt): void
{
    $redis->zAdd('job:queue', $runAt, $jobId);
}

function getReadyJobs(Redis $redis, int $now): array
{
    return $redis->zRangeByScore('job:queue', 0, $now);
}

scheduleJob($redis, 'send-email-7', time() + 60);
$ready = getReadyJobs($redis, time());
```

## Summary

Redis Sets are perfect for tag systems and unique membership checks, while Sorted Sets power leaderboards, priority queues, and any ranked data. Both types are natively supported in PHP through phpredis and Predis, with atomic operations that make them safe for concurrent use.
