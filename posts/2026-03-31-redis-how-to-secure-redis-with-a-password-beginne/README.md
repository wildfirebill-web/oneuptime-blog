# How to Secure Redis with a Password (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Security, Authentication, Password, Beginner, requirepass, ACL

Description: A beginner-friendly guide to securing your Redis instance with a password using requirepass and Redis ACL, preventing unauthorized access.

---

## Why Secure Redis?

By default, Redis has no authentication. Anyone who can connect to the Redis port can read and write all your data. In production environments, this is a serious security risk.

## Method 1: requirepass (Simple Password)

The simplest way to add a password to Redis:

### Setting a Password in redis.conf

```text
# redis.conf
requirepass YourStrongPasswordHere!2026
```

Restart Redis after editing the config file:

```bash
sudo systemctl restart redis-server
```

### Setting a Password at Runtime

```bash
redis-cli CONFIG SET requirepass "YourStrongPasswordHere!2026"
```

### Connecting with a Password

```bash
# Option 1: Authenticate after connecting
redis-cli
AUTH YourStrongPasswordHere!2026

# Option 2: Pass password in the connection command
redis-cli -a "YourStrongPasswordHere!2026"

# Option 3: Use environment variable
export REDIS_PASSWORD="YourStrongPasswordHere!2026"
redis-cli -a "$REDIS_PASSWORD"
```

### What Happens Without Authentication?

```bash
# Connect without password
redis-cli

# Try any command
redis-cli PING
# NOAUTH Authentication required.

# Or try setting a key
redis-cli SET test "value"
# NOAUTH Authentication required.
```

## Choosing a Strong Password

```bash
# Generate a strong random password
openssl rand -base64 32
# Example output: 4kZ8mN2pXqR7sL1vBtYhJwKcFdEuGiAn5oUj9OPlMQ=

# Or use Python
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

Never use simple passwords like "password", "redis123", or "admin".

## Method 2: Redis ACL (Access Control Lists)

Redis 6+ supports ACL for fine-grained access control. You can create users with specific permissions.

### Creating Users with ACL

```bash
# Create a read-only user
ACL SETUSER readonly-user on >readpass ~* &* +@read

# Create a read-write user for a specific key prefix
ACL SETUSER app-user on >apppass ~app:* &* +@all -@dangerous

# Create an admin user
ACL SETUSER admin on >adminpass ~* &* +@all

# List all users
ACL LIST
```

### ACL Rule Syntax

```text
ACL SETUSER <username> <rules>

Rules:
  on / off          - Enable or disable the user
  ><password>       - Set password (e.g., >mypassword)
  ~<pattern>        - Allow access to keys matching pattern (e.g., ~user:*)
  &<pattern>        - Allow Pub/Sub channels (e.g., &* for all)
  +<command>        - Allow command (e.g., +get)
  -<command>        - Deny command
  +@<category>      - Allow all commands in category (e.g., +@read)
  -@<category>      - Deny all commands in category
```

### Viewing ACL Information

```bash
# See your current user
ACL WHOAMI

# See all configured users
ACL LIST

# See details for a specific user
ACL GETUSER app-user

# See all command categories
ACL CAT
```

### Disabling the Default User

```bash
# Disable the default (no-auth) user
ACL SETUSER default off
```

## Connecting with ACL Users

```bash
# Authenticate as a specific user
redis-cli AUTH app-user apppass

# Or in the connection URL
redis-cli -u redis://app-user:apppass@localhost:6379
```

## Node.js with Password Authentication

```javascript
const Redis = require('ioredis');

// With requirepass
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  password: process.env.REDIS_PASSWORD  // Never hardcode!
});

// With ACL user
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  username: 'app-user',   // ACL username
  password: 'apppass'
});

redis.on('error', (err) => console.error('Redis error:', err));
redis.on('connect', () => console.log('Connected to Redis'));
```

## Python with Password Authentication

```python
import redis
import os

# With requirepass
r = redis.Redis(
    host='localhost',
    port=6379,
    password=os.environ.get('REDIS_PASSWORD'),  # From environment
    decode_responses=True
)

# With ACL user (Redis 6+)
r = redis.Redis(
    host='localhost',
    port=6379,
    username='app-user',
    password='apppass',
    decode_responses=True
)

r.ping()  # Will raise AuthenticationError if credentials are wrong
```

## Additional Security: Bind Address

Even with a password, limit which network interfaces Redis listens on:

```text
# redis.conf - only accept connections from localhost
bind 127.0.0.1

# Accept from localhost and a specific internal IP
bind 127.0.0.1 10.0.1.5
```

## Testing Your Security

```bash
# Verify password is required
redis-cli PING
# Should return: NOAUTH Authentication required.

# Verify password works
redis-cli AUTH YourStrongPasswordHere!2026
# Should return: OK

# Verify wrong password fails
redis-cli AUTH wrongpassword
# Should return: WRONGPASS invalid username-password pair
```

## Saving ACL Configuration

```bash
# Save ACL rules to file (so they persist across restarts)
ACL SAVE

# Or configure in redis.conf:
# aclfile /etc/redis/users.acl
```

## Summary

Securing Redis starts with setting a strong password using requirepass in redis.conf or CONFIG SET. For multi-user environments with different access needs, Redis ACL provides fine-grained control - create read-only users for monitoring, restricted users for applications, and admin users for operations. Always store credentials in environment variables or secret managers, never in source code. Pair authentication with binding Redis to a private network interface for defense in depth.
