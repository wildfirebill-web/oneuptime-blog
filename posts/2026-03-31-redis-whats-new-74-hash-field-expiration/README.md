# What Is New in Redis 7.4 (Hash Field Expiration)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Hash, TTL, Expiration, Feature

Description: Redis 7.4 introduced per-field TTL for Hash data types, allowing individual hash fields to expire independently without expiring the entire key.

---

Redis 7.4, released in 2024, delivered a highly anticipated feature: the ability to set expiration on individual fields within a Hash. Previously, TTL could only be set on an entire key - now individual fields can expire independently.

## The Problem Before 7.4

Before 7.4, if you needed some fields of a hash to expire, you had to choose between:

1. Storing each field as a separate string key (wasted memory, harder management)
2. Managing expiry logic in your application code
3. Using a Lua script to check and clean up stale fields

Redis 7.4 solves this natively.

## New Hash Expiry Commands

### HEXPIRE - Set TTL in Seconds

```bash
redis-cli HSET user:123 name "Alice" session "abc123" preferences "{}"

# Expire only the session field after 3600 seconds
redis-cli HEXPIRE user:123 3600 FIELDS 1 session
# 1) (integer) 1
```

### HPEXPIRE - Set TTL in Milliseconds

```bash
redis-cli HPEXPIRE user:123 3600000 FIELDS 1 session
```

### HEXPIREAT - Set Expiry as Unix Timestamp

```bash
redis-cli HEXPIREAT user:123 1800000000 FIELDS 1 session
```

### HPEXPIREAT - Set Expiry as Unix Timestamp in Milliseconds

```bash
redis-cli HPEXPIREAT user:123 1800000000000 FIELDS 1 session
```

## Checking Field TTL

### HTTL - Get TTL in Seconds

```bash
redis-cli HTTL user:123 FIELDS 1 session
# 1) (integer) 3598

redis-cli HTTL user:123 FIELDS 1 name
# 1) (integer) -1  (no expiry)
```

### HPTTL - Get TTL in Milliseconds

```bash
redis-cli HPTTL user:123 FIELDS 1 session
# 1) (integer) 3598000
```

### HEXPIRETIME - Get Absolute Expiry Timestamp

```bash
redis-cli HEXPIRETIME user:123 FIELDS 1 session
# 1) (integer) 1700003600
```

## Removing Field Expiry

Use `HPERSIST` to remove the TTL from a field:

```bash
redis-cli HPERSIST user:123 FIELDS 1 session
# 1) (integer) 1  (field expiry removed)
```

## Practical Use Case: User Sessions in a Hash

Store a user's profile and session together, but expire only the session:

```bash
redis-cli HSET user:123 \
  name "Alice" \
  email "alice@example.com" \
  session_token "tok_abc123" \
  last_login "2024-01-01"

# Session expires in 1 hour, profile is permanent
redis-cli HEXPIRE user:123 3600 FIELDS 1 session_token
```

## Conditional TTL Setting

The `HEXPIRE` command supports condition flags:

```bash
# Set TTL only if field has no existing TTL (NX)
redis-cli HEXPIRE user:123 3600 NX FIELDS 1 session

# Set TTL only if new TTL is greater than current (GT)
redis-cli HEXPIRE user:123 7200 GT FIELDS 1 session
```

## Summary

Redis 7.4's per-field TTL for Hashes is a significant quality-of-life improvement for applications that store mixed-lifetime data in hashes. The `HEXPIRE`, `HTTL`, and `HPERSIST` command families provide fine-grained control over field expiration without requiring separate keys or application-managed cleanup logic.
