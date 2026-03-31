# How to Set Up Cascading Replication in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Replication, Cascading, Architecture, Configuration

Description: Learn how to configure cascading replication in Redis where replicas replicate from other replicas instead of the primary, reducing primary load.

---

Cascading replication (also called chained or hierarchical replication) allows Redis replicas to replicate from other replicas rather than directly from the primary. This reduces the primary's network and CPU load when you need many replicas.

## Architecture Overview

Standard replication (fan-out from primary):

```text
Primary -> Replica-1
Primary -> Replica-2
Primary -> Replica-3
# Primary handles 3 replication streams
```

Cascading replication:

```text
Primary -> Replica-1 -> Replica-2
                     -> Replica-3
# Primary handles 1 stream, Replica-1 fans out to 2 more
```

Cascading is useful when:
- You have 10+ replicas and want to reduce primary load
- Replicas are in different datacenters with limited WAN bandwidth
- The primary handles high write throughput

## Setting Up Cascading Replication

Start the primary:

```bash
# primary: redis.conf
bind 0.0.0.0
port 6379
```

Configure Replica-1 to follow the primary:

```bash
# replica-1: redis.conf
replicaof primary-host 6379
```

Configure Replica-2 and Replica-3 to follow Replica-1:

```bash
# replica-2: redis.conf
replicaof replica-1-host 6379

# replica-3: redis.conf
replicaof replica-1-host 6379
```

Or configure at runtime:

```bash
redis-cli -h replica-2 REPLICAOF replica-1-host 6379
redis-cli -h replica-3 REPLICAOF replica-1-host 6379
```

## Verifying the Topology

Check replication info on each node:

```bash
# On Replica-1
redis-cli -h replica-1 INFO replication
```

```text
role:slave
master_host:primary-host
connected_slaves:2
slave0:ip=replica-2,...
slave1:ip=replica-3,...
```

```bash
# On Replica-2
redis-cli -h replica-2 INFO replication
```

```text
role:slave
master_host:replica-1-host
connected_slaves:0
```

## Intermediate Replica Configuration

Replica-1 acts as both a replica (to the primary) and a primary (to downstream replicas). Ensure it is configured to handle the load:

```bash
# replica-1: redis.conf
# Allow downstream replicas to connect
bind 0.0.0.0
port 6379

# Increase output buffer for downstream replicas
# (handled via CONFIG SET at runtime or redis.conf)

# Replica-1 will buffer replication data for downstream replicas
# Ensure adequate memory
maxmemory 4gb
```

## Replica Read Settings

Downstream replicas are read-only by default:

```bash
redis-cli -h replica-2 CONFIG GET replica-read-only
# replica-read-only:yes
```

If you need replica-2 to also accept writes (not recommended), set `replica-read-only no`, but writes there will not propagate.

## Latency Trade-off

Cascading adds one extra hop of replication latency:

```text
Write on Primary -> Replica-1: ~5ms
Replica-1 -> Replica-2: additional ~5ms
Total lag on Replica-2: ~10ms vs ~5ms direct
```

Monitor lag on each level:

```bash
redis-cli -h replica-1 INFO replication | grep lag
redis-cli -h replica-2 INFO replication | grep lag
```

## Summary

Cascading replication in Redis reduces primary load by having replicas serve as intermediaries for downstream replicas. Configure downstream replicas with `REPLICAOF intermediate-replica-host port`. Trade-off is additional replication latency per hop, so keep cascading depth shallow (2-3 levels maximum) for production use.
