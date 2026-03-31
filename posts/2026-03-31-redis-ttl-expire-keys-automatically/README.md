# How to Use Redis TTL to Expire Keys Automatically

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, TTL, Expiration, Beginner, Cache

Description: Learn how to use Redis TTL to expire keys automatically - covering EXPIRE, EXPIREAT, TTL, PTTL, PERSIST, and expiration patterns.

---

One of Redis's most useful features is automatic key expiration. You can set a Time To Live (TTL) on any key, and Redis will delete it automatically when the time runs out. This is essential for caches, sessions, and temporary data.

## Setting a TTL at Creation

The easiest way is to include the TTL directly in the SET command:

```bash
# Expire in 60 seconds
SET session:abc123 "user_id=42" EX 60

# Expire in 5000 milliseconds (5 seconds)
SET temp_code "984726" PX 5000

# Expire at a specific Unix timestamp
SET promo "10% off" EXAT 1800000000

# Keep the existing TTL (Redis 6.0+)
SET session:abc123 "user_id=99" KEEPTTL
```

## Adding a TTL to an Existing Key

```bash
# Set a key without TTL
SET username "alice"

# Add TTL later (seconds)
EXPIRE username 3600

# Add TTL later (milliseconds)
PEXPIRE username 3600000

# Add TTL at a specific Unix timestamp (seconds)
EXPIREAT username 1800000000

# Add TTL at a specific Unix timestamp (milliseconds)
PEXPIREAT username 1800000000000
```

## Conditional TTL (Redis 7.0+)

```bash
# Only set TTL if key has no TTL
EXPIRE username 3600 NX

# Only set TTL if key already has a TTL
EXPIRE username 3600 XX

# Only set if new TTL is greater than current
EXPIRE username 7200 GT

# Only set if new TTL is less than current
EXPIRE username 60 LT
```

## Checking Remaining Time

```bash
# Check remaining TTL in seconds
TTL username
# (integer) 3542  -> about 59 minutes remaining
# (integer) -1    -> key exists but has no TTL
# (integer) -2    -> key does not exist

# Check remaining TTL in milliseconds
PTTL username
```

## Removing a TTL (Making Key Permanent)

```bash
# Remove TTL, key becomes permanent
PERSIST username
# (integer) 1  -> TTL removed
# (integer) 0  -> key had no TTL or doesn't exist
```

## How Redis Expires Keys

Redis uses two methods to expire keys:

1. **Lazy expiration** - when a key is accessed, Redis checks if it's expired and deletes it then
2. **Active expiration** - every 100ms, Redis randomly samples keys with TTLs and deletes expired ones

This means a key might briefly exist past its TTL if it hasn't been accessed or sampled yet. In practice, expired keys are removed within a few hundred milliseconds.

## Common TTL Patterns

```bash
# Session with rolling TTL (refresh on each access)
SET session:user:42 '{"logged_in":true}' EX 1800

# One-time verification code (10 minutes)
SET verification:email:alice 847291 EX 600

# Rate limiter window (resets every minute)
SET ratelimit:user:42:api 0
EXPIRE ratelimit:user:42:api 60

# Cache with TTL
SET product:42 '{"name":"Widget","price":9.99}' EX 300

# Distributed lock with auto-release
SET lock:payment:order:99 "worker_id_abc" NX EX 30
```

## TTL with Hashes and Other Types

TTL applies to the entire key, not individual fields. Redis 7.4+ adds per-field TTL for hashes:

```bash
# TTL on a hash key (expires the whole hash)
HSET user:42 name "Alice" email "alice@example.com"
EXPIRE user:42 3600

# Redis 7.4+: per-field expiration
HSET session:abc token "xyz" last_seen 1711900000
HEXPIRE session:abc 1800 FIELDS 1 token   # only token field expires
```

## Summary

Redis TTL makes cache management automatic. Use `SET key value EX seconds` for the simplest case, `EXPIRE` to add TTL to existing keys, and `PERSIST` to make a key permanent. Always check TTL with `TTL key` when debugging unexpected key disappearances. Conditional TTL options (NX, XX, GT, LT) in Redis 7.0+ enable precise TTL management without extra round trips.
