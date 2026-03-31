# How to Use ACL WHOAMI in Redis to Get Current Username

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ACL, Security, Authentication, User Management

Description: Learn how to use ACL WHOAMI in Redis to identify which user the current connection is authenticated as, useful for debugging ACL configurations and validating authentication flows.

---

## Overview

`ACL WHOAMI` returns the username of the currently authenticated user for the connection issuing the command. It is a simple diagnostic command that helps you verify which user is active on a connection -- particularly useful when debugging ACL rules, testing authentication flows, or confirming that a client library has authenticated correctly.

```mermaid
flowchart LR
    A[Client connects] --> B[Optionally authenticates with AUTH]
    B --> C[Issues ACL WHOAMI]
    C --> D[Redis returns current username]
    D --> E{Which user?}
    E -- Not authenticated --> F[Returns 'default']
    E -- Authenticated as alice --> G[Returns 'alice']
```

## Syntax

```redis
ACL WHOAMI
```

Returns a bulk string with the current username.

## Basic Usage

### Check current user without authentication

When a connection has not explicitly authenticated, it operates as the `default` user:

```redis
ACL WHOAMI
```

```text
"default"
```

### Check current user after authentication

```redis
AUTH alice secretpassword
ACL WHOAMI
```

```text
"alice"
```

### Check user in redis-cli

```bash
redis-cli -u redis://alice:secretpassword@127.0.0.1:6379 ACL WHOAMI
```

```text
"alice"
```

## Practical Use Cases

### Validate client authentication in application code

In application startup, call `ACL WHOAMI` after connecting to confirm the client authenticated as the expected user:

```redis
# After connecting and authenticating
ACL WHOAMI
```

```text
"app_service"
```

If this returns `"default"` when you expected `"app_service"`, the client did not authenticate.

### Confirm permissions before running sensitive commands

```redis
ACL WHOAMI
```

```text
"readonly_user"
```

Knowing the current user, you can anticipate which commands are permitted without trial and error.

### Debug multi-user connection pools

When using connection pools that authenticate as different users, `ACL WHOAMI` lets you verify which user is active on a given connection from a pool.

## Relationship with ACL GETUSER

`ACL WHOAMI` tells you who you are. `ACL GETUSER` tells you what a user is allowed to do:

```redis
# Who am I?
ACL WHOAMI
```

```text
"alice"
```

```redis
# What can alice do?
ACL GETUSER alice
```

```text
1) "flags"
2) 1) "on"
3) "passwords"
4) (empty array)
5) "commands"
6) "+@read"
7) "keys"
8) "~cache:*"
...
```

## Summary

`ACL WHOAMI` is a lightweight introspection command that returns the username for the current connection. Unauthenticated connections return `"default"`. Use it in application startup checks to verify authentication succeeded, in debugging sessions to confirm which user is active, and in scripts to conditionally branch based on user identity. It requires no arguments and always succeeds as long as the server is reachable.
