# What Does "ERR invalid password" Mean in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Authentication, Error, Security, Debugging

Description: Understand the Redis "ERR invalid password" error, what causes it, and how to fix authentication issues in Redis configuration and client code.

---

The `ERR invalid password` error in Redis occurs when you provide an incorrect password with the `AUTH` command. It indicates the password supplied does not match the one configured in `requirepass` or the ACL user's password.

## What Causes the Error

```bash
127.0.0.1:6379> AUTH wrongpassword
(error) ERR invalid password
```

This error can happen in three scenarios:

1. The password is simply wrong - a typo or stale credential
2. The password was changed in `redis.conf` or via `CONFIG SET` and the client still uses the old value
3. In Redis 6+ ACL mode, the user exists but the password is incorrect

## Fixing the Error

**Step 1: Verify the configured password**

Check `redis.conf` for the `requirepass` directive:

```bash
grep requirepass /etc/redis/redis.conf
```

Or check the running configuration:

```bash
127.0.0.1:6379> CONFIG GET requirepass
1) "requirepass"
2) "your-current-password"
```

Note: `CONFIG GET requirepass` only works if you are already authenticated or if no password is set.

**Step 2: Authenticate correctly**

```bash
127.0.0.1:6379> AUTH yourcorrectpassword
OK
```

In Redis 6+ with ACL, specify the username too:

```bash
127.0.0.1:6379> AUTH myuser yourcorrectpassword
OK
```

## Setting or Changing the Password

```bash
# Set a password at runtime
127.0.0.1:6379> CONFIG SET requirepass "newstrongpassword"
OK

# After this, all subsequent commands require authentication
127.0.0.1:6379> AUTH newstrongpassword
OK
```

To persist the change, update `redis.conf`:

```text
requirepass newstrongpassword
```

## Fixing Client Configuration

Update the password in your application's Redis connection string or environment variables.

```bash
# .env file
REDIS_PASSWORD=newstrongpassword
```

```python
import redis

r = redis.Redis(
    host='127.0.0.1',
    port=6379,
    password='newstrongpassword',
    decode_responses=True,
)
r.ping()
```

```javascript
const Redis = require('ioredis');
const client = new Redis({
  host: '127.0.0.1',
  port: 6379,
  password: process.env.REDIS_PASSWORD,
});
```

## ACL-Based Authentication (Redis 6+)

If you are using ACL users, the error may be due to the wrong password for that specific user:

```bash
# Check existing ACL users
127.0.0.1:6379> ACL LIST
1) "user default on nopass ~* &* +@all"
2) "user appuser on #<hash> ~* +@read +@write"

# Reset a user's password
127.0.0.1:6379> ACL SETUSER appuser >newpassword
OK
```

## Summary

The "ERR invalid password" error means the password sent with `AUTH` does not match the configured `requirepass` value or the ACL user's password. Check `redis.conf` or `CONFIG GET requirepass` for the current password, update your client credentials, and use `ACL SETUSER` when managing users with Redis 6+ ACLs.
