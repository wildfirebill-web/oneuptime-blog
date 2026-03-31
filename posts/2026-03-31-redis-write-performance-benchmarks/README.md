# How to Write Redis Performance Benchmarks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Performance, Benchmarking, Throughput, Latency, Testing

Description: Write meaningful Redis performance benchmarks to measure throughput, latency, and pipeline efficiency using redis-benchmark, Python, and custom workload scripts.

---

## What to Measure

Good Redis benchmarks measure:
- **Throughput** - operations per second (ops/sec)
- **Latency** - p50, p95, p99 response times
- **Pipeline efficiency** - benefit of batching commands
- **Connection overhead** - pool vs. single connection

## Using redis-benchmark

Redis ships with `redis-benchmark` for baseline measurements:

```bash
# Run default benchmark (100 clients, 100000 requests)
redis-benchmark -h 127.0.0.1 -p 6379

# Test SET with 10000 requests, 50 clients, 3 threads
redis-benchmark -n 10000 -c 50 --threads 3 SET mykey myvalue

# Test GET with data size 100 bytes
redis-benchmark -n 50000 -c 100 -d 100 GET mykey

# Test pipelining (batch 16 commands)
redis-benchmark -n 100000 -c 50 -P 16 SET key value

# Test specific commands only
redis-benchmark -t set,get,incr,lpush,rpush,sadd,hset -n 100000
```

## Python Benchmark Framework

```python
import redis
import time
import statistics
from contextlib import contextmanager

r = redis.Redis(decode_responses=True)

@contextmanager
def timer():
    start = time.perf_counter()
    yield
    elapsed = time.perf_counter() - start
    print(f"Elapsed: {elapsed:.4f}s")

def benchmark_operation(func, iterations=10000, label=""):
    latencies = []
    for _ in range(iterations):
        start = time.perf_counter()
        func()
        latencies.append((time.perf_counter() - start) * 1000)

    p50 = statistics.median(latencies)
    p95 = sorted(latencies)[int(len(latencies) * 0.95)]
    p99 = sorted(latencies)[int(len(latencies) * 0.99)]
    ops_sec = iterations / (sum(latencies) / 1000)

    print(f"[{label}] ops/sec={ops_sec:.0f} p50={p50:.3f}ms p95={p95:.3f}ms p99={p99:.3f}ms")

# SET benchmark
r.set("bench_key", "value")
benchmark_operation(lambda: r.set("bench_key", "value"), label="SET")

# GET benchmark
benchmark_operation(lambda: r.get("bench_key"), label="GET")

# HSET benchmark
benchmark_operation(lambda: r.hset("bench_hash", "field", "value"), label="HSET")

# INCR benchmark
r.set("bench_counter", 0)
benchmark_operation(lambda: r.incr("bench_counter"), label="INCR")
```

## Pipeline vs. Non-Pipeline Comparison

```python
def benchmark_without_pipeline(iterations=1000):
    start = time.perf_counter()
    for i in range(iterations):
        r.set(f"key:{i}", f"value:{i}")
    elapsed = time.perf_counter() - start
    print(f"Without pipeline: {elapsed:.4f}s ({iterations/elapsed:.0f} ops/sec)")

def benchmark_with_pipeline(iterations=1000, batch_size=100):
    start = time.perf_counter()
    for batch_start in range(0, iterations, batch_size):
        pipe = r.pipeline()
        for i in range(batch_start, min(batch_start + batch_size, iterations)):
            pipe.set(f"key:{i}", f"value:{i}")
        pipe.execute()
    elapsed = time.perf_counter() - start
    print(f"With pipeline (batch={batch_size}): {elapsed:.4f}s ({iterations/elapsed:.0f} ops/sec)")

benchmark_without_pipeline(1000)
benchmark_with_pipeline(1000, batch_size=100)
```

## Benchmarking Connection Pool Sizes

```python
def benchmark_pool_size(pool_size: int, iterations=5000, concurrency=50):
    import threading

    pool = redis.ConnectionPool(max_connections=pool_size)
    rc = redis.Redis(connection_pool=pool, decode_responses=True)

    def worker():
        for _ in range(iterations // concurrency):
            rc.set("key", "value")

    start = time.perf_counter()
    threads = [threading.Thread(target=worker) for _ in range(concurrency)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    elapsed = time.perf_counter() - start

    print(f"Pool size={pool_size}: {elapsed:.4f}s ({iterations/elapsed:.0f} ops/sec)")

for size in [10, 50, 100, 200]:
    benchmark_pool_size(size)
```

## Benchmarking Data Structure Operations

```python
def benchmark_sorted_set():
    r.delete("bench:leaderboard")
    members = {f"user:{i}": float(i * 10) for i in range(1000)}

    # Bulk insert
    start = time.perf_counter()
    r.zadd("bench:leaderboard", members)
    print(f"ZADD 1000 members: {(time.perf_counter()-start)*1000:.2f}ms")

    # Range query
    start = time.perf_counter()
    for _ in range(1000):
        r.zrevrange("bench:leaderboard", 0, 9)
    elapsed = time.perf_counter() - start
    print(f"ZREVRANGE top-10 x1000: {elapsed:.4f}s ({1000/elapsed:.0f} ops/sec)")

benchmark_sorted_set()
```

## Latency Histogram with histogram-based Output

```bash
redis-benchmark -n 100000 -c 50 --csv -t set,get | tee benchmark_results.csv
```

Parse the CSV to analyze:

```python
import csv

with open("benchmark_results.csv") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(f"{row['test']}: {row['rps']} ops/sec, avg={row['avg_latency_ms']}ms")
```

## Summary

Redis performance benchmarks should cover throughput, latency percentiles, and pipelining efficiency to understand real-world performance. `redis-benchmark` provides a quick baseline, while custom Python benchmarks measure application-specific workloads and compare optimization strategies like connection pooling and command batching. Always benchmark with production-representative data sizes and concurrency levels.
