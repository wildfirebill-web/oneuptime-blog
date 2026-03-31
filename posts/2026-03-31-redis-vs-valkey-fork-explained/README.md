# Redis vs Valkey: The Fork Explained

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Valkey, Open Source, Fork, Comparison

Description: Understand the Redis vs Valkey fork - why it happened, what is different, how compatible they are, and how to decide which one to use for your production workloads.

---

In March 2024, Redis Ltd. changed Redis's license from BSD-3-Clause to SSPL/RSALv2. Within days, major cloud providers and contributors forked Redis 7.2 as Valkey, donated it to the Linux Foundation, and continued development under the BSD-3-Clause license. This guide explains the fork and helps you decide which project fits your needs.

## Why the Fork Happened

Redis 7.2 was the last version released under the permissive BSD-3-Clause license. When Redis Ltd. announced the license change for 7.4+:

- Amazon Web Services, Google Cloud, Oracle, Ericsson, and others objected
- These companies formed a group and forked Redis 7.2.4 as Valkey
- The Linux Foundation accepted Valkey as a new project in March 2024
- The Linux Foundation provides neutral governance and prevents future license changes

## Technical Compatibility

Valkey 7.2 is a drop-in replacement for Redis 7.2:

```bash
# Valkey uses the same wire protocol
redis-cli -p 6379 PING          # Works against Valkey
valkey-cli -p 6379 PING         # Also works

# Data migration: Redis RDB files work with Valkey
redis-cli --rdb dump.rdb        # Export from Redis
valkey-server --rdbfilename dump.rdb  # Import in Valkey
```

All existing Redis clients (ioredis, redis-py, Jedis, StackExchange.Redis) work with Valkey without modification.

## Feature Divergence After the Fork

Since the fork, both projects have evolved:

```text
Feature                    | Redis 7.4+           | Valkey 8.x
---------------------------|----------------------|------------------
License                    | SSPL / RSALv2        | BSD-3-Clause
Governance                 | Redis Ltd.           | Linux Foundation
Multi-threading I/O        | Limited              | Enhanced in 8.x
New data structures        | Redis 7.4 additions  | Independent roadmap
Cloud provider support     | Commercial licensing | AWS, GCP native
Community fork             | No                   | Yes (Linux Foundation)
```

Valkey 8.0 introduced enhanced multi-threaded I/O handling, which improves throughput on multi-core systems.

## Comparing Performance

Valkey 8.x shows performance improvements over Redis 7.2 due to multi-threading:

```bash
# Benchmark comparison (rough example on 8-core server)
# Redis 7.2: ~400k ops/sec SET
# Valkey 8.x: ~600k ops/sec SET (multi-threaded)

redis-benchmark -h localhost -p 6379 -t set,get -n 1000000 -c 50
valkey-benchmark -h localhost -p 6379 -t set,get -n 1000000 -c 50
```

## Choosing Between Redis and Valkey

```text
Choose Redis if:
- You need Redis Stack modules (RediSearch, RedisJSON, RedisTimeSeries)
- You use Redis Enterprise cloud offering
- You require commercial support from Redis Ltd.
- You are already using Redis 7.4+ features

Choose Valkey if:
- You prioritize open-source licensing (BSD)
- You use AWS (ElastiCache Valkey), GCP (Memorystore Valkey), or other managed options
- You want Linux Foundation governance
- You need better multi-threaded performance
- You are starting a new project and want to avoid license risk
```

## Migrating from Redis to Valkey

Migration is straightforward for Redis 7.2-compatible workloads:

```bash
# 1. Create RDB snapshot from Redis
redis-cli BGSAVE
# Wait for RDB to complete

# 2. Copy RDB file to Valkey server
scp /var/lib/redis/dump.rdb valkey-server:/var/lib/valkey/

# 3. Start Valkey with the RDB file
valkey-server --dir /var/lib/valkey

# 4. Verify data
valkey-cli DBSIZE
valkey-cli KEYS "*" | head -10
```

## Summary

Valkey is a BSD-licensed fork of Redis 7.2, governed by the Linux Foundation, created in response to Redis Ltd.'s SSPL/RSALv2 licensing change. It is wire-compatible with Redis 7.2, supported natively by major cloud providers, and includes performance improvements in version 8.x. For new projects, Valkey offers open-source licensing with no restrictions. For existing Redis deployments, migration is straightforward given full protocol and data format compatibility.
