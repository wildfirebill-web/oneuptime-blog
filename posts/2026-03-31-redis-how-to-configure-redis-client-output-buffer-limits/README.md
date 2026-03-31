# How to Configure Redis Client Output Buffer Limits

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Performance, Configuration, Memory Management

Description: Learn how to configure Redis client output buffer limits to prevent memory exhaustion and protect your server from slow or lagging clients.

---

## What Are Client Output Buffers?

Redis maintains an output buffer for each connected client. When Redis sends data to a client, it temporarily stores that data in the buffer until the client reads it. If the client is slow to consume data - for example, a slow consumer in a Pub/Sub channel or a large SCAN result - the buffer can grow unboundedly and consume significant memory.

Redis provides configurable limits to control this behavior through the `client-output-buffer-limit` directive.

## Buffer Limit Categories

Redis applies output buffer limits to three categories of clients:

- `normal` - Regular clients issuing commands
- `slave` (replica) - Replica clients during replication
- `pubsub` - Clients subscribed to Pub/Sub channels

## Configuration Syntax

The configuration directive takes the following form:

```text
client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
```

- `hard limit` - If a client buffer exceeds this size, Redis disconnects the client immediately
- `soft limit` - If the buffer stays above this size for longer than `soft seconds`, Redis disconnects the client
- `0` means no limit

## Default Configuration

Redis ships with these defaults:

```text
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

Normal clients have no limit by default. Replicas get a hard limit of 256 MB and pubsub clients get 32 MB.

## Setting Limits in redis.conf

Open your `redis.conf` file and add or modify the limits:

```bash
# Normal clients - set a 64MB hard limit
client-output-buffer-limit normal 64mb 16mb 60

# Replica clients - increase for high-write scenarios
client-output-buffer-limit replica 512mb 128mb 120

# Pub/Sub clients
client-output-buffer-limit pubsub 32mb 8mb 60
```

## Setting Limits at Runtime

You can apply changes without restarting using the `CONFIG SET` command:

```bash
redis-cli CONFIG SET client-output-buffer-limit "normal 64mb 16mb 60"
```

To inspect the current setting:

```bash
redis-cli CONFIG GET client-output-buffer-limit
```

## Monitoring Buffer Usage

Use the `CLIENT LIST` command to inspect buffer sizes for connected clients:

```bash
redis-cli CLIENT LIST
```

Look for the `omem` field in the output, which shows the output buffer size in bytes:

```text
id=3 addr=127.0.0.1:52345 ... omem=12345 ...
```

You can also use `INFO clients` to get aggregate stats:

```bash
redis-cli INFO clients
```

Sample output:

```text
connected_clients:42
blocked_clients:0
tracking_clients:0
clients_in_timeout_table:0
client_recent_max_input_buffer:20504
client_recent_max_output_buffer:0
```

## Diagnosing Buffer-Related Disconnections

If clients are being disconnected unexpectedly, check the Redis log for messages like:

```text
[32235] 12 Mar 09:05:24.456 # Client id=5 addr=127.0.0.1:56789 fd=12 name= age=120 idle=0 flags=P db=0 sub=10 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=1024 omem=33554432 events=r cmd=subscribe scheduled to be closed ASAP for overcoming of output buffer limits.
```

The key fields here are `oll` (output list length) and `omem` (output memory usage).

## Tuning for Pub/Sub Workloads

Pub/Sub workloads are especially prone to buffer overflow when publishers produce messages faster than subscribers consume them. For high-throughput Pub/Sub, increase the limits:

```text
client-output-buffer-limit pubsub 128mb 32mb 120
```

Also consider rate-limiting publishers at the application layer to match subscriber throughput.

## Tuning for Replication

During an initial full sync, a replica receives a large RDB snapshot. If this snapshot is large and the network is slow, the replica buffer can grow quickly. Increase the replica hard limit to accommodate large datasets:

```text
client-output-buffer-limit replica 1gb 256mb 120
```

## Best Practices

- Monitor `omem` in `CLIENT LIST` regularly
- Set non-zero limits for normal clients in multi-tenant or public-facing environments
- Increase replica limits when initial sync RDB files are large
- Use slow consumers monitoring to detect lagging Pub/Sub subscribers
- Restart or reload config with `CONFIG REWRITE` to persist runtime changes

## Summary

Redis client output buffer limits protect your server from memory exhaustion caused by slow or lagging clients. By configuring hard and soft limits for normal, replica, and pubsub client classes, you can ensure stability under load. Monitoring `omem` in `CLIENT LIST` and Redis logs will help you tune these values appropriately for your workload.
