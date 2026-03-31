# How to Troubleshoot Redis AUTH Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Authentication, AUTH, Security, Troubleshooting

Description: Fix Redis AUTH errors by verifying password configuration, ACL users, TLS settings, and ensuring clients send credentials in the correct format.

---

## Types of Redis AUTH Errors

Redis returns AUTH-related errors in several forms:

```text
WRONGPASS invalid username-password pair or user is disabled.
NOAUTH Authentication required.
ERR Client sent AUTH, but no password is set. Did you mean ACL SETUSER with >password?
```

Each indicates a different misconfiguration.

## NOAUTH - Server Requires Auth but Client Did Not Send It

If Redis has a password configured (`requirepass`) but the client connects without calling AUTH, every command returns:

```text
NOAUTH Authentication required.
```

Fix: configure your client to send credentials on connect.

```bash
# redis-cli
redis-cli -h localhost -a mysecretpassword PING

# Or authenticate after connecting
redis-cli -h localhost
127.0.0.1:6379> AUTH mysecretpassword
OK
```

In Python:

```python
import redis
r = redis.Redis(host='localhost', port=6379, password='mysecretpassword')
r.ping()
```

In Node.js:

```javascript
const client = require('redis').createClient({
  url: 'redis://:mysecretpassword@localhost:6379'
});
```

## WRONGPASS - Incorrect Password or Disabled User

This error appears when:

1. The password is wrong
2. The ACL user is disabled
3. The username does not exist

Check the server-side password:

```bash
redis-cli CONFIG GET requirepass
```

If using ACL:

```bash
redis-cli ACL LIST
```

Look for the relevant user and verify the password hash and status:

```text
user default on #<sha256hash> ~* &* +@all
user myapp on #<sha256hash> ~cache:* +GET +SET
```

`on` means enabled, `off` means disabled. If the user is `off`:

```bash
redis-cli ACL SETUSER myapp on
```

Reset a password:

```bash
redis-cli ACL SETUSER myapp >newpassword
```

## ERR - Client Sent AUTH but No Password is Set

If you send AUTH to a Redis server with no `requirepass` set and not using ACL users:

```text
ERR Client sent AUTH, but no password is set. Did you mean ACL SETUSER with >password?
```

Either remove the AUTH call from your client or set a password on the server:

```bash
redis-cli CONFIG SET requirepass mynewpassword
```

## Step - Check ACL User Configuration

Redis 6.0+ uses ACL for fine-grained access control. The default user is configured by `requirepass` in `redis.conf`, but you can also use the ACL system directly.

Create and configure a user:

```bash
# Create user 'appuser' with password, access to keys starting with 'app:', read/write
redis-cli ACL SETUSER appuser on >secretpass ~app:* +@read +@write

# Verify
redis-cli ACL GETUSER appuser
```

Test the user:

```bash
redis-cli -h localhost --user appuser --pass secretpass PING
```

## Step - Authenticate with Username in Redis 6.0+

If using ACL users, clients must send both username and password:

```bash
# AUTH username password syntax
redis-cli -h localhost AUTH appuser secretpass
```

In Python with redis-py:

```python
import redis
r = redis.Redis(
    host='localhost',
    port=6379,
    username='appuser',
    password='secretpass'
)
```

In ioredis:

```javascript
const client = new Redis({
  host: 'localhost',
  port: 6379,
  username: 'appuser',
  password: 'secretpass'
});
```

## Sentinel Authentication

If using Redis Sentinel, both the Sentinel nodes and the Redis instances may require authentication. Configure Sentinel to authenticate to Redis:

```text
# In sentinel.conf
sentinel auth-pass mymaster mysecretpassword
sentinel auth-user mymaster appuser
```

Also protect Sentinel itself:

```text
requirepass sentinelpassword
```

## TLS and Certificate Authentication

If Redis is configured with TLS (`tls-port`), connection issues may appear as AUTH errors if the certificate chain is invalid. Check:

```bash
redis-cli --tls --cacert /etc/redis/ca.crt -h localhost -p 6380 PING
```

## Summary

Redis AUTH errors fall into three categories: the client is not sending credentials (NOAUTH), the credentials are wrong or the user is disabled (WRONGPASS), or AUTH is being sent to a server with no password configured (ERR). Check server-side configuration with `CONFIG GET requirepass` and `ACL LIST`, then ensure clients pass the correct username and password. In Redis 6.0+ with ACL, always include the username in the AUTH call.
