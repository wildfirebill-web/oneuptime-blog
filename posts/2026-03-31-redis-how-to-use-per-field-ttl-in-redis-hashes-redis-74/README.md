# How to Use Per-Field TTL in Redis Hashes (Redis 7.4+)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Hashes, Ttl, Redis 7.4, Expiration

Description: Use Redis 7.4 per-field expiration commands to set individual TTLs on Hash fields, enabling fine-grained expiration without key-level workarounds.

---

## What Is Per-Field TTL in Hashes

Before Redis 7.4, TTL could only be set on entire keys, not individual fields within a Hash. To expire individual fields, developers resorted to workarounds like separate keys or periodic cleanup jobs.

Redis 7.4 introduced per-field expiration for Hashes with new commands:
- `HEXPIRE key seconds field [field ...]`
- `HEXPIREAT key unix-time-seconds field [field ...]`
- `HPEXPIRE key milliseconds field [field ...]`
- `HPEXPIREAT key unix-time-ms field [field ...]`
- `HTTL key field [field ...]`
- `HPTTL key field [field ...]`
- `HEXPIRETIME key field [field ...]`
- `HPEXPIRETIME key field [field ...]`
- `HPERSIST key field [field ...]`

## Setting Field Expiration

```bash
# Create a hash with user session data
redis-cli HSET user:session:123 \
  userId 42 \
  accessToken "tok_abc123" \
  refreshToken "ref_xyz789" \
  lastSeen "2024-01-01T00:00:00Z"

# Set 1-hour TTL on the access token field only
redis-cli HEXPIRE user:session:123 3600 FIELDS 1 accessToken

# Set 30-day TTL on the refresh token field
redis-cli HEXPIRE user:session:123 2592000 FIELDS 1 refreshToken

# Fields without expiration persist indefinitely
redis-cli HTTL user:session:123 FIELDS 4 userId accessToken refreshToken lastSeen
# Output: -1, 3600, 2592000, -1
# -1 means no expiration
```

## Setting Exact Expiration Time

```bash
# Expire a field at a specific Unix timestamp
redis-cli HEXPIREAT user:session:123 1735689600 FIELDS 1 accessToken

# Millisecond precision
redis-cli HPEXPIRE user:session:123 3600000 FIELDS 1 accessToken

# Unix timestamp in milliseconds
redis-cli HPEXPIREAT user:session:123 1735689600000 FIELDS 1 accessToken
```

## Checking Field TTL

```bash
# Get remaining TTL in seconds for multiple fields
redis-cli HTTL user:session:123 FIELDS 2 accessToken refreshToken
# Output:
# 1) (integer) 3598
# 2) (integer) 2591998

# Get TTL in milliseconds
redis-cli HPTTL user:session:123 FIELDS 1 accessToken

# Get absolute expiration time as Unix timestamp
redis-cli HEXPIRETIME user:session:123 FIELDS 1 accessToken
```

## Removing Field Expiration

```bash
# Make a field permanent (remove its expiration)
redis-cli HPERSIST user:session:123 FIELDS 1 accessToken

# Verify: returns -1 (no expiration)
redis-cli HTTL user:session:123 FIELDS 1 accessToken
```

## Practical Example: Token Management

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: process.env.REDIS_HOST || 'localhost' });

async function createUserSession(userId, tokens) {
  const sessionKey = `user:session:${userId}`;

  // Store all session data in one Hash
  await redis.hset(sessionKey, {
    userId: userId.toString(),
    accessToken: tokens.access,
    refreshToken: tokens.refresh,
    createdAt: Date.now().toString(),
  });

  // Set field-level TTLs
  await redis.call(
    'HEXPIRE', sessionKey, tokens.accessExpiresIn,
    'FIELDS', '1', 'accessToken'
  );
  await redis.call(
    'HEXPIRE', sessionKey, tokens.refreshExpiresIn,
    'FIELDS', '1', 'refreshToken'
  );

  return sessionKey;
}

async function getSessionToken(userId, tokenType) {
  const sessionKey = `user:session:${userId}`;

  // HGET returns null if field has expired
  const token = await redis.hget(sessionKey, tokenType);

  if (!token) {
    // Check if key itself exists but field expired
    const exists = await redis.exists(sessionKey);
    if (exists) {
      throw new Error(`${tokenType} has expired`);
    }
    throw new Error('Session not found');
  }

  return token;
}

async function rotateAccessToken(userId, newAccessToken, expiresIn = 3600) {
  const sessionKey = `user:session:${userId}`;

  // Update the field
  await redis.hset(sessionKey, 'accessToken', newAccessToken);

  // Set new expiration on the field
  await redis.call('HEXPIRE', sessionKey, expiresIn, 'FIELDS', '1', 'accessToken');
}
```

## Multi-Field Expiration

```bash
# Set different TTLs on multiple fields at once
redis-cli HEXPIRE user:cache:42 300 FIELDS 3 profile preferences avatar

# Check all at once
redis-cli HTTL user:cache:42 FIELDS 3 profile preferences avatar
# 1) (integer) 299
# 2) (integer) 299
# 3) (integer) 299
```

## Return Value Codes

`HEXPIRE`, `HEXPIREAT`, `HPEXPIRE`, and `HPEXPIREAT` return:
- `2` - field does not exist in the hash
- `1` - expiration was set successfully
- `0` - conditions not met (e.g., NX/XX/GT/LT flags)
- `-1` - field does not have a TTL (for HTTL/HPTTL)
- `-2` - field does not exist (for HTTL/HPTTL)

```bash
redis-cli HEXPIRE user:123 3600 FIELDS 2 existingField nonExistentField
# 1) (integer) 1   <- existingField: success
# 2) (integer) 2   <- nonExistentField: does not exist
```

## Summary

Redis 7.4 per-field TTL in Hashes eliminates the need for workarounds when different fields of the same object have different expiration requirements. Use `HEXPIRE` to set field-level TTLs in seconds, `HTTL` to query remaining time, and `HPERSIST` to remove expiration from a field. This is particularly useful for session management where access and refresh tokens have different lifetimes but belong to the same logical session object.
