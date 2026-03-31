# Redis Timeout Configuration Best Practices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Timeout, Configuration, Best Practice, Performance

Description: Learn how to configure Redis timeouts correctly - covering connect timeouts, command timeouts, idle timeouts, and blocking command timeouts for production reliability.

---

Misconfigured timeouts are a common source of Redis-related production incidents. Too short and you get false failures; too long and a slow Redis blocks your application threads. This guide covers every timeout type in Redis and how to configure each one appropriately.

## Connect Timeout vs. Command Timeout

These two are frequently confused but serve different purposes:

```python
import redis

client = redis.Redis(
    host='redis.example.com',
    port=6379,
    socket_connect_timeout=2,  # Time to establish TCP connection
    socket_timeout=5           # Time to wait for a command response
)
```

- **Connect timeout**: How long to wait for the TCP handshake. Keep this short (1-3 seconds).
- **Command timeout**: How long to wait for a Redis response. Set this based on your slowest expected command.

## Server-Side Client Timeout

In `redis.conf`, you can close idle client connections server-side:

```text
timeout 300
```

This closes connections that have been idle for 300 seconds. This is useful for reclaiming resources from abandoned connections. Setting it to `0` disables it.

## TCP Keep-Alive Timeout

The `tcp-keepalive` setting controls how often Redis sends TCP keep-alive probes:

```text
tcp-keepalive 60
```

This sends a keep-alive every 60 seconds. If the client has died without closing the connection, it will be detected and cleaned up.

## Blocking Command Timeouts

Commands like `BLPOP`, `BRPOP`, and `XREAD BLOCK` accept a timeout argument:

```bash
# Block for up to 5 seconds waiting for a list element
BLPOP mylist 5

# Block indefinitely (0 means no timeout - dangerous in production)
BLPOP mylist 0
```

Never use `0` (infinite block) in production. Use a reasonable timeout and handle the nil response when it expires:

```python
result = client.blpop('queue', timeout=10)
if result is None:
    # Timeout expired, no data available
    continue
key, value = result
```

## Lua Script Timeouts

Long-running Lua scripts can block the entire Redis server. The `lua-time-limit` setting controls how long a script can run before Redis starts responding to other clients with a `BUSY` error:

```text
lua-time-limit 5000
```

This is in milliseconds. The default is 5000ms (5 seconds). Once this limit is exceeded, Redis allows `SCRIPT KILL` or `SHUTDOWN NOSAVE` commands only.

## Cluster Node Timeout

In Redis Cluster mode, `cluster-node-timeout` determines when a node is considered failed:

```text
cluster-node-timeout 15000
```

Set this to at least 15 seconds in production. Too short causes false failovers during network hiccups; too long delays actual failover.

## Sentinel Timeouts

For Redis Sentinel, configure how quickly it detects a down master:

```text
sentinel down-after-milliseconds mymaster 30000
sentinel failover-timeout mymaster 180000
```

- `down-after-milliseconds`: Time before a master is considered unreachable (30s is a safe default)
- `failover-timeout`: Maximum time for the failover process

## Choosing Timeout Values

Use these guidelines:

```text
socket_connect_timeout: 1-3 seconds
socket_timeout: 3-10 seconds (higher for heavy workloads)
server-side timeout: 300-600 seconds
tcp-keepalive: 60 seconds
lua-time-limit: 5000ms
cluster-node-timeout: 15000ms
```

Always test your timeout values under load before deploying to production.

## Summary

Redis has multiple timeout settings that operate at different layers - client-side connect and command timeouts, server-side idle timeouts, and cluster/sentinel failover timeouts. Configure each one deliberately based on your workload characteristics. Excessively short timeouts cause false failures while excessively long timeouts allow cascading failures to persist.
