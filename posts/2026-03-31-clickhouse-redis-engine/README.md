# How to Use Redis Table Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Redis, Storage Engine, Cache, Integration

Description: Learn how to use the Redis table engine in ClickHouse to read key-value data from Redis, join it with analytical tables, and use Redis as a low-latency lookup source.

---

The `Redis` table engine connects ClickHouse to a Redis key-value store and lets you query Redis data as a ClickHouse table. It maps Redis keys to the primary key column and Redis hash fields (or string values) to other columns. The engine is useful for enriching ClickHouse queries with real-time data stored in Redis, such as user sessions, feature flags, rate-limit counters, or real-time inventory levels.

## Prerequisites

ClickHouse must be able to reach the Redis server on its port (default 6379). Ensure the Redis user (if ACLs are enabled) has `GET` and `HGETALL` permissions.

## Creating a Redis Engine Table

The Redis engine supports two storage modes:
- **Simple mode**: each key maps to a single string value.
- **Hash mode**: each key maps to a Redis hash (multiple fields).

```sql
-- Hash mode: key = user_id, fields = username, plan, country, last_seen
CREATE TABLE redis_user_cache
(
    user_id   UInt64,    -- Redis key
    username  String,    -- Redis hash field
    plan      String,    -- Redis hash field
    country   String,    -- Redis hash field
    last_seen DateTime   -- Redis hash field
)
ENGINE = Redis(
    'redis-host:6379',
    0,                   -- Redis database number (0-15)
    'secret_password'    -- Redis AUTH password (empty string if no auth)
)
PRIMARY KEY user_id;
```

## Reading From Redis

```sql
SELECT user_id, username, plan, country, last_seen
FROM redis_user_cache
WHERE user_id IN (1001, 1002, 1003);
```

```text
user_id  username  plan     country  last_seen
1001     alice     premium  US       2024-06-15 10:30:00
1002     bob       pro      DE       2024-06-15 09:45:00
1003     carol     free     JP       2024-06-15 08:00:00
```

## Inserting Data Into Redis

The Redis engine supports `INSERT`, which writes to Redis as hash keys.

```sql
INSERT INTO redis_user_cache VALUES
    (2001, 'dave',   'pro',     'GB', '2024-06-15 11:00:00'),
    (2002, 'eve',    'premium', 'AU', '2024-06-15 11:05:00'),
    (2003, 'frank',  'free',    'CA', '2024-06-15 11:10:00');
```

Each row becomes a Redis hash at key `2001`, `2002`, `2003` with fields `username`, `plan`, `country`, `last_seen`.

## Updating a Row

Insert a row with the same primary key to overwrite the existing Redis hash fields.

```sql
-- Dave upgraded to premium
INSERT INTO redis_user_cache VALUES
    (2001, 'dave', 'premium', 'GB', now());
```

## Joining Redis Data With ClickHouse Analytics

The most common use case is enriching analytical queries with live data from Redis.

```sql
-- Enrich ClickHouse event data with Redis user cache
SELECT
    e.event_date,
    r.plan,
    r.country,
    count()          AS events,
    uniq(e.user_id)  AS unique_users,
    sum(e.revenue)   AS revenue
FROM daily_events AS e
JOIN redis_user_cache AS r ON e.user_id = r.user_id
WHERE e.event_date = today()
GROUP BY e.event_date, r.plan, r.country
ORDER BY revenue DESC
LIMIT 10;
```

## Real-Time Feature Flags From Redis

```sql
CREATE TABLE redis_feature_flags
(
    flag_name   String,    -- Redis key
    enabled     UInt8,     -- Redis hash field
    rollout_pct UInt8,     -- Redis hash field
    updated_by  String     -- Redis hash field
)
ENGINE = Redis('redis-host:6379', 0, '')
PRIMARY KEY flag_name;

-- Check which features are currently enabled
SELECT flag_name, enabled, rollout_pct, updated_by
FROM redis_feature_flags
ORDER BY flag_name;
```

```text
flag_name            enabled  rollout_pct  updated_by
dark_mode            1        100          ops-team
new_checkout         1        25           product-team
ai_recommendations   0        0            ml-team
```

## Rate-Limit Counter Lookup

```sql
CREATE TABLE redis_rate_limits
(
    key       String,   -- Redis key, e.g., "ratelimit:user:1001:/api/search"
    count     UInt32,   -- current hit count
    expires   DateTime  -- TTL timestamp
)
ENGINE = Redis('redis-host:6379', 1, 'secret')
PRIMARY KEY key;

-- Check if a user is close to their rate limit
SELECT
    key,
    count,
    expires,
    count >= 100 AS is_throttled
FROM redis_rate_limits
WHERE key LIKE 'ratelimit:user:1001:%'
  AND expires > now();
```

## Session Store Integration

```sql
CREATE TABLE redis_sessions
(
    session_id  String,
    user_id     UInt64,
    ip_address  String,
    created_at  DateTime,
    expires_at  DateTime,
    is_valid    UInt8
)
ENGINE = Redis('redis-host:6379', 2, 'session_pass')
PRIMARY KEY session_id;

-- Find all active sessions for a user
SELECT session_id, ip_address, created_at, expires_at
FROM redis_sessions
WHERE user_id = 1001
  AND is_valid = 1
  AND expires_at > now()
ORDER BY created_at DESC;
```

## Deleting Data From Redis

```sql
-- Delete expired sessions from Redis via ClickHouse
DELETE FROM redis_sessions
WHERE expires_at < now();
```

## Full Scan Limitations

A full table scan reads all keys from Redis, which can be slow on large Redis instances. Prefer point lookups or small `IN` lists.

```sql
-- Slow - full Redis SCAN
SELECT count() FROM redis_user_cache WHERE plan = 'premium';

-- Fast - direct key lookup
SELECT *
FROM redis_user_cache
WHERE user_id IN (1001, 1002, 1003);
```

## Using Named Collections for Credentials

```xml
<!-- config.d/named_collections.xml -->
<yandex>
  <named_collections>
    <redis_prod>
      <host>redis-host</host>
      <port>6379</port>
      <password>secret_password</password>
      <db_index>0</db_index>
    </redis_prod>
  </named_collections>
</yandex>
```

```sql
CREATE TABLE redis_cache
(
    user_id  UInt64,
    username String,
    plan     String
)
ENGINE = Redis(redis_prod)
PRIMARY KEY user_id;
```

## Redis Cluster Support

For Redis Cluster, specify any node in the cluster - ClickHouse follows redirects automatically.

```sql
CREATE TABLE redis_cluster_cache
(
    key   String,
    value String
)
ENGINE = Redis(
    'redis-cluster-node-01:6379',
    0,
    'auth_password'
)
PRIMARY KEY key;
```

## Limitations

- Only key-based lookup (`WHERE primary_key = ...` or `IN`) is efficient; range scans are not supported.
- Full table scans use Redis `SCAN` which is slow on large keyspaces.
- No ClickHouse-side compression or columnar storage.
- Redis TTL (key expiry) is not reflected in ClickHouse queries until the key is physically expired.
- Not suitable for analytical workloads requiring aggregation over many keys.

## Summary

The `Redis` engine provides a live bridge between Redis key-value data and ClickHouse analytics. Use it for low-latency lookups of sessions, feature flags, rate limits, and real-time reference data. For best performance, always access Redis through primary key lookups using `WHERE key = ...` or `IN (...)` rather than full scans. For analytical workloads requiring aggregation over large Redis keyspaces, replicate the data into a MergeTree table periodically.
