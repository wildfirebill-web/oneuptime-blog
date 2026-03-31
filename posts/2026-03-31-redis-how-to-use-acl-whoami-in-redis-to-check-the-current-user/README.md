# How to Use ACL WHOAMI in Redis to Check the Current User

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ACL, Security, Command, Access Control

Description: Learn how to use ACL WHOAMI in Redis to identify which ACL user the current connection is authenticated as, useful for debugging and connection verification.

---

## What Is ACL WHOAMI

`ACL WHOAMI` returns the username of the current Redis connection. This is useful for verifying that your application is authenticating as the expected ACL user, debugging permission issues, and confirming user context after `AUTH` calls.

```text
ACL WHOAMI
```

Returns the current username as a bulk string.

## Basic Usage

```bash
# Default unauthenticated user
ACL WHOAMI
# "default"

# After authenticating as a specific user
AUTH myuser mypassword
ACL WHOAMI
# "myuser"
```

The default user in Redis (when no ACL file is configured) is named `default`.

## Practical Example in Python

```python
import redis

# Connect without explicit auth (uses 'default' user)
client = redis.Redis(host='localhost', port=6379, decode_responses=True)
username = client.acl_whoami()
print(f"Current user: {username}")  # default

# Connect with specific user
app_client = redis.Redis(
    host='localhost',
    port=6379,
    username='app_service',
    password='strongpassword',
    decode_responses=True
)
username = app_client.acl_whoami()
print(f"App user: {username}")  # app_service
```

## Practical Example in Node.js

```javascript
const { createClient } = require('redis');

// Default connection
const client = createClient();
await client.connect();

let user = await client.aclWhoAmI();
console.log(`Current user: ${user}`); // default

// Authenticated connection
const authClient = createClient({
  username: 'app_service',
  password: 'strongpassword'
});
await authClient.connect();

user = await authClient.aclWhoAmI();
console.log(`Auth user: ${user}`); // app_service
```

## Practical Example in Go

```go
package main

import (
    "context"
    "fmt"
    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()

    // Default client
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })

    user, err := rdb.ACLWhoAmI(ctx).Result()
    if err != nil {
        panic(err)
    }
    fmt.Printf("Current user: %s\n", user)

    // Authenticated client
    authRdb := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Username: "app_service",
        Password: "strongpassword",
    })

    user, err = authRdb.ACLWhoAmI(ctx).Result()
    if err != nil {
        panic(err)
    }
    fmt.Printf("Auth user: %s\n", user)
}
```

## Connection Verification Pattern

Use `ACL WHOAMI` at startup to verify your application is using the correct identity:

```python
import redis
import sys

EXPECTED_USER = 'api_service'

client = redis.Redis(
    host='localhost',
    port=6379,
    username=EXPECTED_USER,
    password='mypassword',
    decode_responses=True
)

def verify_connection():
    try:
        current_user = client.acl_whoami()
        if current_user != EXPECTED_USER:
            print(f"ERROR: Expected '{EXPECTED_USER}' but connected as '{current_user}'")
            sys.exit(1)
        print(f"Connection verified: authenticated as '{current_user}'")
        return True
    except redis.AuthenticationError as e:
        print(f"Authentication failed: {e}")
        sys.exit(1)

verify_connection()
```

## Using ACL WHOAMI After AUTH

You can switch users on an existing connection using `AUTH`:

```bash
# Start as default
ACL WHOAMI
# "default"

# Switch to another user
AUTH admin adminpassword
ACL WHOAMI
# "admin"

# Switch back if needed
AUTH default defaultpassword
ACL WHOAMI
# "default"
```

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Switch users programmatically
def switch_user(client, username, password):
    client.auth(password, username=username)
    current = client.acl_whoami()
    print(f"Now authenticated as: {current}")
    return current

switch_user(client, 'admin', 'adminpassword')
switch_user(client, 'readonly', 'readpassword')
```

## Debugging ACL Permission Issues

When a command fails with permission errors, use `ACL WHOAMI` to confirm which user the connection is using:

```python
import redis

client = redis.Redis(
    host='localhost',
    port=6379,
    username='limited_user',
    password='limitedpass',
    decode_responses=True
)

try:
    client.flushall()
except redis.ResponseError as e:
    current_user = client.acl_whoami()
    print(f"Permission denied for user '{current_user}': {e}")
    print("Check ACL rules with: ACL GETUSER", current_user)
```

## Health Check Integration

```python
import redis

def redis_health_check(expected_user='default'):
    try:
        client = redis.Redis(host='localhost', port=6379, decode_responses=True)
        pong = client.ping()
        user = client.acl_whoami()

        return {
            'status': 'healthy',
            'ping': pong,
            'user': user,
            'user_match': user == expected_user
        }
    except Exception as e:
        return {
            'status': 'unhealthy',
            'error': str(e)
        }

status = redis_health_check(expected_user='api_service')
print(status)
```

## Summary

`ACL WHOAMI` returns the username of the currently authenticated Redis connection. Use it at application startup to verify correct user configuration, during debugging to confirm identity when permission errors occur, and in health checks to validate authentication state. It is the fastest way to check which ACL identity a connection is operating under.
