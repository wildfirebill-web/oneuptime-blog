# How to Plan Redis Capacity and Sizing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Capacity Planning, Performance

Description: Learn how to plan Redis capacity by estimating memory needs, connection counts, throughput requirements, and growth projections for your workload.

---

Correct Redis capacity planning prevents unexpected OOM errors, latency spikes, and costly over-provisioning. This guide walks through the key dimensions of sizing a Redis deployment.

## Memory Sizing

Redis stores all data in RAM. Start by measuring your dataset size:

```bash
# Current memory usage
redis-cli INFO memory | grep -E "used_memory_human|used_memory_rss_human"

# Estimate key sizes
redis-cli MEMORY USAGE mykey
redis-cli MEMORY DOCTOR
```

Calculate memory per key type:

```text
String:    ~64 bytes overhead + value size
Hash:      ~200 bytes base + (field_name + value) * num_fields
List:      ~140 bytes base + 16 bytes per element
Set:       ~200 bytes base + 16 bytes per member
Sorted Set: ~200 bytes base + 32 bytes per member
```

For a dataset with 1M string keys averaging 200 bytes each:

```text
1,000,000 keys x (64 + 200) bytes = ~264 MB data
Add 30% overhead for Redis internal structures: ~343 MB
Add 20% headroom for growth: ~412 MB
Recommended maxmemory: 512 MB
```

## Estimating Connection Requirements

Each connected client consumes approximately 20KB of memory:

```bash
redis-cli INFO clients
```

```text
# Calculate connection memory
connections=500
memory_per_conn_kb=20
total_conn_mb=$((connections * memory_per_conn_kb / 1024))
# 500 connections = ~10 MB
```

Set appropriate limits:

```bash
redis-cli CONFIG SET maxclients 1000
```

## Throughput and CPU Planning

Benchmark your expected operations per second:

```bash
# Built-in benchmark
redis-benchmark -h 127.0.0.1 -p 6379 -n 100000 -c 50 -q

# Benchmark specific commands
redis-benchmark -t set,get,lpush -n 100000 -q
```

Sample output:

```text
SET: 125000.00 requests per second
GET: 140000.00 requests per second
LPUSH: 118000.00 requests per second
```

Redis is single-threaded per core for commands. Plan for:
- 1 CPU core per 100K ops/sec for simple commands
- 2 cores for complex operations (SORT, ZUNIONSTORE)
- Extra cores for persistence (AOF/RDB background saves)

## Persistence Overhead

```text
RDB: Uses fork() - needs 2x memory momentarily during save
AOF rewrite: Also uses fork() + write buffer

Recommendation for 2 GB dataset with AOF:
- Allocate 5 GB RAM (2 GB data + 2 GB fork + 1 GB headroom)
```

## Network Sizing

```bash
# Check current network throughput
redis-cli INFO stats | grep -E "total_net_input|total_net_output"

# Estimate bandwidth
# 1000 ops/sec x 100 byte avg payload = 100 KB/s = 0.8 Mbps
# Plan for 10x peak: 8 Mbps minimum NIC capacity
```

## Capacity Planning Template

```text
Dataset size:        _____ GB
Growth rate/month:   _____ %
6-month projection:  _____ GB
Memory (2x for fork): _____ GB
Connection memory:   _____ MB
Recommended RAM:     _____ GB

Peak ops/sec:        _____
Recommended vCPUs:   _____
```

## Summary

Redis capacity planning requires sizing for current dataset memory, connection overhead, fork() requirements for persistence, and projected growth. Use `redis-benchmark` for throughput baselines, `MEMORY USAGE` for per-key analysis, and always include at least 20-30% headroom above calculated requirements.
