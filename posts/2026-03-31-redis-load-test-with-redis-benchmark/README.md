# How to Load Test Redis with redis-benchmark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Load Testing, redis-benchmark, Performance, Throughput, Latency

Description: Use redis-benchmark to load test Redis with customized workloads, connection counts, data sizes, and pipeline batching to identify throughput limits and latency bottlenecks.

---

## What is redis-benchmark

`redis-benchmark` is a tool bundled with Redis that simulates concurrent clients sending commands and measures throughput (requests/sec) and latency. It is not meant to replace application-level load testing but gives a quick baseline for Redis server capacity.

## Basic Usage

Run the default benchmark suite:

```bash
redis-benchmark
```

This runs GET, SET, INCR, LPUSH, RPUSH, SADD, HSET, SPOP, MSET across 50 clients with 100,000 requests each.

## Common Flags

```bash
# -h host, -p port
redis-benchmark -h 127.0.0.1 -p 6379

# -n total requests
redis-benchmark -n 500000

# -c concurrent clients
redis-benchmark -c 100

# -d value size in bytes
redis-benchmark -d 1024

# -t test specific commands
redis-benchmark -t set,get,incr

# -P pipeline depth (batch N commands per request)
redis-benchmark -P 16

# --threads number of benchmark threads
redis-benchmark --threads 4

# -q quiet mode (summary only)
redis-benchmark -q

# --csv output results as CSV
redis-benchmark --csv -t set,get

# --latency-history show latency over time
redis-benchmark --latency-history
```

## Benchmarking SET and GET

```bash
# 500k SET operations, 100 clients, 100 byte values
redis-benchmark -n 500000 -c 100 -d 100 -t set -q

# 500k GET operations
redis-benchmark -n 500000 -c 100 -d 100 -t get -q
```

Example output:

```text
SET: 452315.16 requests per second, p50=0.167 msec
GET: 487804.88 requests per second, p50=0.151 msec
```

## Pipelining Benchmark

Pipelining batches multiple commands in a single TCP roundtrip. Measure the speedup:

```bash
# Without pipelining
redis-benchmark -n 100000 -c 50 -t set -q

# With pipeline depth of 16
redis-benchmark -n 100000 -c 50 -P 16 -t set -q

# With pipeline depth of 32
redis-benchmark -n 100000 -c 50 -P 32 -t set -q
```

## Benchmarking Different Data Sizes

```bash
for size in 64 256 1024 4096 16384; do
    echo -n "Value size ${size} bytes: "
    redis-benchmark -n 100000 -c 50 -d $size -t set -q
done
```

## Benchmarking HSET and Sorted Sets

```bash
# HSET with 10 fields
redis-benchmark -n 100000 -c 50 -t hset -q

# Custom command: ZADD
redis-benchmark -n 100000 -c 50 -q \
    ZADD myset __rand_int__ __rand_int__

# Custom command: ZRANGEBYSCORE
redis-benchmark -n 100000 -c 50 -q \
    ZRANGEBYSCORE myset 0 100
```

## Custom Command Benchmarks

```bash
# Benchmark a custom Lua script
redis-benchmark -n 50000 -c 50 -q \
    EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 key:__rand_int__ value:__rand_int__

# Benchmark PUBLISH
redis-benchmark -n 100000 -c 50 -q PUBLISH mychannel "test message"
```

## Latency Distribution

Get detailed percentile latency stats:

```bash
redis-benchmark -n 100000 -c 100 -t set --latency-history -i 1
```

Output shows latency evolution per second, useful for spotting latency spikes.

## Benchmarking Over TLS

```bash
redis-benchmark -n 100000 -c 50 \
    --tls \
    --cert /path/to/redis.crt \
    --key /path/to/redis.key \
    --cacert /path/to/ca.crt \
    -t set,get -q
```

## Parsing CSV Output

```bash
redis-benchmark --csv -t set,get,incr,hset -n 100000 -c 50 | tee results.csv
```

```python
import csv
import sys

with open("results.csv") as f:
    reader = csv.reader(f)
    for row in reader:
        test, rps, avg_latency_ms, min_ms, p50, p95, p99, max_ms = row
        print(f"{test}: {float(rps):.0f} rps | p50={p50}ms p99={p99}ms")
```

## Interpreting Results

- If throughput is CPU-bound on the server, scale with Redis Cluster or increase `io-threads`
- High p99 latency relative to p50 indicates GC pauses, network jitter, or slow commands
- Pipelining typically gives 5-10x throughput improvement but increases per-request latency

```bash
# Enable io-threads for multi-core throughput
redis-server --io-threads 4 --io-threads-do-reads yes
```

## Summary

`redis-benchmark` provides fast baselines for Redis throughput and latency across standard command types. Adjust client count, request count, data size, and pipeline depth to simulate your workload. CSV output enables trend analysis and regression detection across deployments. Always combine `redis-benchmark` results with application-level load tests using real query patterns.
