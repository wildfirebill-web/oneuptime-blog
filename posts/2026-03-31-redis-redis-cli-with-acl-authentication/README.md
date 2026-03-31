# How to Use Redis CLI with ACL Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis CLI, ACL, Authentication, Security

Description: Learn how to authenticate to Redis using ACL usernames and passwords in redis-cli, including TLS connections and scripting with credentials.

---

## Redis ACL Authentication Overview

Redis 6.0 introduced Access Control Lists (ACLs), which allow multiple users with different passwords and permissions. The legacy `AUTH password` command still works for the default user, but ACL authentication uses the two-argument form: `AUTH username password`.

The Redis CLI supports ACL authentication via the `-u` flag or inline `AUTH` commands.

## Authenticating with Username and Password

Use the `-u` flag with a Redis URI to connect with ACL credentials:

```bash
redis-cli -u "redis://appuser:secretpassword@127.0.0.1:6379"
```

Alternatively, pass the username and password as separate flags using `--user` and `--pass` (Redis CLI 7.0+):

```bash
redis-cli -h 127.0.0.1 -p 6379 --user appuser --pass secretpassword
```

Or connect first and then authenticate interactively:

```bash
redis-cli -h 127.0.0.1 -p 6379
127.0.0.1:6379> AUTH appuser secretpassword
OK
```

## Checking the Current User

After connecting, verify which user is active:

```text
127.0.0.1:6379> ACL WHOAMI
"appuser"
```

## Using the Default User

The default user is named `default`. If you set a password for it:

```bash
redis-cli -h 127.0.0.1 -p 6379 -a "defaultpassword"
```

This is equivalent to:

```bash
redis-cli -h 127.0.0.1 -p 6379 --user default --pass defaultpassword
```

## Scripting with ACL Credentials

When running non-interactive commands in scripts, include credentials inline:

```bash
redis-cli -u "redis://appuser:secretpassword@127.0.0.1:6379" GET mykey
```

For shell scripts using environment variables:

```bash
#!/bin/bash

REDIS_USER="${REDIS_USER:-appuser}"
REDIS_PASS="${REDIS_PASS:-secretpassword}"
REDIS_HOST="${REDIS_HOST:-127.0.0.1}"
REDIS_PORT="${REDIS_PORT:-6379}"

redis-cli -u "redis://${REDIS_USER}:${REDIS_PASS}@${REDIS_HOST}:${REDIS_PORT}" \
  GET mykey
```

Avoid putting passwords directly in shell scripts or command history. Use environment variables or a secrets manager.

## Connecting with ACL Over TLS

For TLS-enabled Redis with ACL authentication:

```bash
redis-cli -h redis.example.com -p 6380 \
  --tls \
  --cacert /etc/ssl/redis-ca.crt \
  --user appuser \
  --pass secretpassword
```

With client certificate authentication (mTLS):

```bash
redis-cli -h redis.example.com -p 6380 \
  --tls \
  --cacert /etc/ssl/redis-ca.crt \
  --cert /etc/ssl/client.crt \
  --key /etc/ssl/client.key \
  --user appuser \
  --pass secretpassword
```

## Testing ACL Permissions with DRYRUN

The `ACL DRYRUN` command lets you check whether a user has permission to execute a specific command without actually running it:

```text
127.0.0.1:6379> ACL DRYRUN appuser SET foo bar
OK

127.0.0.1:6379> ACL DRYRUN readonly_user SET foo bar
(error) User readonly_user has no permissions to run the 'set' command
```

This is useful for auditing permissions without generating errors in logs.

## Viewing Permissions for the Current User

After authenticating as a user, inspect what commands and keys are accessible:

```text
127.0.0.1:6379> ACL GETUSER appuser
1) "flags"
2) 1) "on"
3) "passwords"
4) 1) "..."
5) "commands"
6) "+@read +@write -@dangerous"
7) "keys"
8) "~cache:* ~session:*"
9) "channels"
10) "&*"
```

## Handling Authentication Errors

Common ACL authentication errors:

```text
(error) WRONGPASS invalid username-password pair or user is disabled.
```

This means either the username does not exist, the password is wrong, or the user is disabled. Check the ACL list:

```bash
redis-cli -u "redis://admin:adminpass@127.0.0.1:6379" ACL LIST
```

```text
(error) NOPERM this user has no permissions to run the 'acl|list' command
```

This means the connected user lacks permission for the `ACL LIST` command. Connect as a user with admin privileges.

## Rotating Passwords via CLI

To rotate a user's password without downtime, add the new password before removing the old one:

```text
127.0.0.1:6379> ACL SETUSER appuser >newpassword
OK
# Update application config to use new password
127.0.0.1:6379> ACL SETUSER appuser <oldpassword
OK
```

## Summary

Redis CLI supports ACL authentication through the `-u` URI flag, `--user`/`--pass` flags, and inline `AUTH username password` commands. For secure scripting, store credentials in environment variables and use TLS for encrypted transport. The `ACL DRYRUN` command is a valuable tool for testing permissions without generating noise in logs or triggering access denials in production.
