# How to Build a Guild/Clan Management System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Gaming, Guild, Hash, Set

Description: Build a scalable guild and clan management system using Redis sets, hashes, and sorted sets to handle memberships, roles, contributions, and rankings efficiently.

---

Guilds and clans are social hubs in multiplayer games. They need fast membership lookups, contribution tracking, and guild rankings. Redis data structures map naturally to these requirements without a heavy relational schema.

## Guild Metadata

Store guild info in a hash:

```bash
HSET guild:42 name "Iron Legion" leader_id 1001 level 5 created_at 1700000000
HGET guild:42 name
# "Iron Legion"
```

## Membership Management

Track members with a set for O(1) lookups and easy iteration:

```bash
SADD guild:42:members 1001 1002 1003 1004
SISMEMBER guild:42:members 1003
# 1 (member)
SCARD guild:42:members
# 4
```

Link the player back to their guild:

```bash
SET player:1001:guild_id 42
```

## Member Roles

Use a hash to store each member's role:

```bash
HSET guild:42:roles 1001 leader 1002 officer 1003 member 1004 member
HGET guild:42:roles 1002
# "officer"
```

## Contribution Tracking

Track each member's weekly contribution score with a sorted set:

```bash
ZINCRBY guild:42:contributions 500 1001
ZINCRBY guild:42:contributions 300 1002
ZREVRANGE guild:42:contributions 0 -1 WITHSCORES
```

Reset contributions each week using TTL or a scheduled key rotation:

```python
import redis
r = redis.Redis()

def reset_weekly_contributions(guild_id):
    old_key = f"guild:{guild_id}:contributions"
    archive_key = f"guild:{guild_id}:contributions:archive:{get_week()}"
    r.rename(old_key, archive_key)
    r.expire(archive_key, 604800 * 4)  # Keep 4 weeks
```

## Guild Rankings

Maintain a global guild leaderboard by XP:

```bash
ZADD guilds:leaderboard 15000 42
ZADD guilds:leaderboard 12000 55
ZREVRANK guilds:leaderboard 42
# 0 (top guild)
```

Update guild XP when members earn it:

```python
def award_guild_xp(guild_id, amount):
    r.zincrby("guilds:leaderboard", amount, guild_id)
    r.hincrby(f"guild:{guild_id}", "total_xp", amount)
```

## Guild Chat History

Store the last N messages in a list:

```bash
LPUSH guild:42:chat "1001:Hello everyone!"
LTRIM guild:42:chat 0 99
LRANGE guild:42:chat 0 9
```

## Leaving and Disbanding

Remove a member cleanly:

```python
def leave_guild(player_id, guild_id):
    pipe = r.pipeline()
    pipe.srem(f"guild:{guild_id}:members", player_id)
    pipe.hdel(f"guild:{guild_id}:roles", player_id)
    pipe.zrem(f"guild:{guild_id}:contributions", player_id)
    pipe.delete(f"player:{player_id}:guild_id")
    pipe.execute()
```

## Summary

Redis makes guild systems fast and simple by mapping members to sets, roles to hashes, and rankings to sorted sets. Pipeline operations keep member removal atomic and efficient, while sorted sets give you instant top-contributor and top-guild rankings without expensive database queries.
