# How to Use REPLICAOF in Redis to Configure Replication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Replication, High Availability, Replicaof, Standby

Description: Learn how to use REPLICAOF in Redis to configure a standalone instance as a replica of a master, enabling replication without cluster mode.

---

## What Is REPLICAOF?

`REPLICAOF` is a Redis command that configures a Redis instance to become a replica (formerly called slave) of another instance. When set, the replica continuously synchronizes data from the designated master, maintaining an up-to-date copy of all keyspace data.

`REPLICAOF` replaces the deprecated `SLAVEOF` command introduced in Redis 1.0. Both commands are functionally identical, but `REPLICAOF` is the preferred modern form.

## Basic Syntax

```text
REPLICAOF host port
REPLICAOF NO ONE
```

- `REPLICAOF host port` - make this instance a replica of the specified master
- `REPLICAOF NO ONE` - promote this instance to master (stop replication)

## Setting Up Replication

### On the Replica Instance

```bash
# Configure this instance to replicate from master at 192.168.1.10:6379
redis-cli -h 127.0.0.1 -p 6380 REPLICAOF 192.168.1.10 6379
# Returns: OK
```

Redis immediately begins the synchronization process - first a full sync (RDB transfer) then ongoing incremental replication.

### Verifying Replication Status

```bash
redis-cli -h 127.0.0.1 -p 6380 INFO replication
```

```text
role:slave
master_host:192.168.1.10
master_port:6379
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_repl_offset:98765
slave_priority:100
slave_read_only:1
```

## Configuring Replication via redis.conf

For persistent configuration, set replication in the config file instead of using the runtime command:

```text
replicaof 192.168.1.10 6379
replica-read-only yes
replica-serve-stale-data yes
```

## Promoting a Replica to Master

When a master fails and you want to promote a replica to take over:

```bash
redis-cli -h 127.0.0.1 -p 6380 REPLICAOF NO ONE
# Returns: OK
```

After running this:
- The replica stops syncing from the old master
- It becomes a standalone read-write master
- Clients can start writing to it

```bash
# Verify promotion
redis-cli -h 127.0.0.1 -p 6380 INFO replication | grep role
# role:master
```

## Replication with Authentication

If the master requires authentication, configure the replica with a password:

```bash
# In redis.conf on the replica
masterauth your-master-password

# Or at runtime
redis-cli -h 127.0.0.1 -p 6380 CONFIG SET masterauth your-master-password
redis-cli -h 127.0.0.1 -p 6380 REPLICAOF 192.168.1.10 6379
```

## Read-Only Replicas

By default, replicas are read-only. Write attempts return an error:

```bash
redis-cli -h 127.0.0.1 -p 6380 SET foo bar
# READONLY You can't write against a read only replica.
```

To allow writes on a replica (not recommended for production):

```bash
redis-cli -h 127.0.0.1 -p 6380 CONFIG SET replica-read-only no
```

## Monitoring Replication Lag

```bash
# Check replication offset difference
redis-cli -h 192.168.1.10 -p 6379 INFO replication | grep master_repl_offset
# master_repl_offset:98765

redis-cli -h 127.0.0.1 -p 6380 INFO replication | grep slave_repl_offset
# slave_repl_offset:98765  (no lag if equal to master)
```

## Python Example

```python
import redis

master = redis.Redis(host='192.168.1.10', port=6379, decode_responses=True)
replica = redis.Redis(host='127.0.0.1', port=6380, decode_responses=True)

# Set up replication
replica.replicaof('192.168.1.10', 6379)

# Check status
info = replica.info('replication')
print(f"Role: {info['role']}")
print(f"Master: {info.get('master_host')}:{info.get('master_port')}")
print(f"Link status: {info.get('master_link_status')}")
print(f"Replication offset: {info.get('slave_repl_offset')}")

# Promote to master
# replica.replicaof('NO', 'ONE')
```

## Chained Replication

Redis supports replica-of-replica (cascading replication), useful for reducing load on the master:

```bash
# Master: 192.168.1.10:6379
# Replica1: 192.168.1.11:6379 (replicates from master)
# Replica2: 192.168.1.12:6379 (replicates from Replica1)

redis-cli -h 192.168.1.11 -p 6379 REPLICAOF 192.168.1.10 6379
redis-cli -h 192.168.1.12 -p 6379 REPLICAOF 192.168.1.11 6379
```

## REPLICAOF vs CLUSTER REPLICATE

| Feature | REPLICAOF | CLUSTER REPLICATE |
|---|---|---|
| Mode | Standalone (non-cluster) | Cluster mode only |
| Reference | IP + port | Node ID |
| Use case | Simple replication | Cluster HA |
| Failover | Manual (or Sentinel) | Automatic |

## Summary

`REPLICAOF` is the standard command for configuring Redis replication in standalone (non-cluster) deployments. It allows you to set up a replica with a single command and promote it to master using `REPLICAOF NO ONE` when needed. Combine `REPLICAOF` with Redis Sentinel for automated failover in production environments, or use Redis Cluster for a fully distributed solution.
