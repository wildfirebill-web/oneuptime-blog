# How to Write Redis Performance Benchmarks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Performance, Benchmarking, Redis-Benchmark, Python, Node.js

Description: Learn how to write meaningful Redis performance benchmarks using redis-benchmark, custom Python scripts, and proper methodology to measure throughput and latency.

---

## Why Benchmark Redis

Redis is fast, but performance varies significantly based on:
- Network latency (local vs. remote Redis)
- Pipeline usage vs. individual commands
- Value size and data type
- Number of concurrent connections
- Key eviction policy
- Server hardware and CPU pinning

Benchmarking before and after changes gives you concrete numbers rather than intuition.

## Using the Built-in redis-benchmark Tool

Redis ships with `redis-benchmark`, which is the fastest way to establish baseline numbers:

```bash
# Basic benchmark - 100,000 requests, 50 concurrent clients
redis-benchmark -h localhost -p 6379 -n 100000 -c 50

# Benchmark specific commands
redis-benchmark -h localhost -p 6379 -t set,get,incr,lpush,rpush -n 100000

# Benchmark with pipelining (16 commands per pipeline)
redis-benchmark -h localhost -p 6379 -n 100000 -P 16

# Benchmark with larger values (1KB)
redis-benchmark -h localhost -p 6379 -n 100000 -d 1024 -t set,get

# Output in CSV format
redis-benchmark -h localhost -p 6379 -n 100000 --csv
```

Example output:

```text
====== SET ======
  100000 requests completed in 0.54 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1

99.00% <= 1 milliseconds
99.50% <= 1 milliseconds
100.00% <= 2 milliseconds
185185.19 requests per second
```

## Custom Python Benchmark

For benchmarking application-specific patterns, write a custom benchmark:

```python
import redis
import time
import statistics
import json
from typing import Callable

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def benchmark(name: str, fn: Callable, iterations: int = 10000):
    latencies = []
    start_total = time.perf_counter()

    for i in range(iterations):
        start = time.perf_counter()
        fn(i)
        elapsed = (time.perf_counter() - start) * 1000  # ms
        latencies.append(elapsed)

    total = time.perf_counter() - start_total
    latencies.sort()

    print(f"\n{name} ({iterations} iterations):")
    print(f"  Throughput:   {iterations / total:.0f} ops/sec")
    print(f"  Avg latency:  {statistics.mean(latencies):.3f} ms")
    print(f"  P50 latency:  {latencies[int(len(latencies) * 0.50)]:.3f} ms")
    print(f"  P95 latency:  {latencies[int(len(latencies) * 0.95)]:.3f} ms")
    print(f"  P99 latency:  {latencies[int(len(latencies) * 0.99)]:.3f} ms")
    print(f"  Max latency:  {max(latencies):.3f} ms")

# Benchmark: simple SET
def bench_set(i):
    r.set(f"key:{i}", f"value:{i}")

# Benchmark: SET with TTL
def bench_setex(i):
    r.setex(f"key:{i}", 3600, f"value:{i}")

# Benchmark: JSON value serialization
sample_data = {"user_id": 12345, "name": "Alice", "roles": ["admin", "editor"], "score": 9850}
json_payload = json.dumps(sample_data)

def bench_set_json(i):
    r.set(f"user:{i}", json_payload)

def bench_get_json(i):
    raw = r.get(f"user:{i % 1000}")
    if raw:
        json.loads(raw)

# Benchmark: Hash operations
def bench_hset(i):
    r.hset(f"hash:{i}", mapping={"f1": "v1", "f2": "v2", "f3": "v3"})

# Run all benchmarks
benchmark("SET simple", bench_set)
benchmark("SETEX with TTL", bench_setex)
benchmark("SET JSON payload", bench_set_json)
benchmark("GET+deserialize JSON", bench_get_json)
benchmark("HSET 3 fields", bench_hset)
```

## Pipeline vs. Individual Commands

Pipelining is one of the most impactful optimizations. Benchmark both:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379)
N = 10000

# Benchmark without pipeline
start = time.perf_counter()
for i in range(N):
    r.set(f"key:{i}", f"value:{i}")
individual_time = time.perf_counter() - start

# Benchmark with pipeline
start = time.perf_counter()
pipe = r.pipeline()
for i in range(N):
    pipe.set(f"key:{i}", f"value:{i}")
pipe.execute()
pipeline_time = time.perf_counter() - start

print(f"Individual: {N/individual_time:.0f} ops/sec ({individual_time:.2f}s)")
print(f"Pipeline:   {N/pipeline_time:.0f} ops/sec ({pipeline_time:.2f}s)")
print(f"Speedup:    {individual_time/pipeline_time:.1f}x")
```

## Node.js Benchmark

```javascript
const Redis = require('ioredis');
const client = new Redis();

async function benchmark(name, fn, iterations = 10000) {
  const latencies = [];
  const start = process.hrtime.bigint();

  for (let i = 0; i < iterations; i++) {
    const opStart = process.hrtime.bigint();
    await fn(i);
    const opEnd = process.hrtime.bigint();
    latencies.push(Number(opEnd - opStart) / 1e6); // Convert to ms
  }

  const total = Number(process.hrtime.bigint() - start) / 1e9;
  latencies.sort((a, b) => a - b);

  console.log(`\n${name} (${iterations} iterations):`);
  console.log(`  Throughput: ${Math.round(iterations / total)} ops/sec`);
  console.log(`  P50: ${latencies[Math.floor(iterations * 0.5)].toFixed(3)} ms`);
  console.log(`  P95: ${latencies[Math.floor(iterations * 0.95)].toFixed(3)} ms`);
  console.log(`  P99: ${latencies[Math.floor(iterations * 0.99)].toFixed(3)} ms`);
}

async function run() {
  await benchmark('SET', async (i) => {
    await client.set(`key:${i}`, `value:${i}`);
  });

  await benchmark('GET', async (i) => {
    await client.get(`key:${i % 1000}`);
  });

  await benchmark('HSET', async (i) => {
    await client.hset(`hash:${i}`, 'field1', 'value1', 'field2', 'value2');
  });

  client.disconnect();
}

run().catch(console.error);
```

## Benchmarking Best Practices

```text
1. Warm up before measuring: run 1000 ops before starting the timer
2. Run multiple iterations and report percentiles, not just averages
3. Match the benchmark environment to production (same network, same hardware)
4. Benchmark with realistic payload sizes (not just 3-byte values)
5. Test with realistic concurrency levels
6. Isolate variables: test one change at a time
7. Measure both throughput (ops/sec) and latency (P50, P95, P99)
8. Run benchmarks multiple times and compare results for consistency
```

Example warmup pattern:

```python
# Warmup phase
for i in range(1000):
    r.set(f"warmup:{i}", "x")

# Now benchmark
benchmark("SET after warmup", bench_set)
```

## Summary

Effective Redis benchmarking combines the built-in redis-benchmark tool for baseline numbers with custom scripts that match your actual access patterns. Always measure percentile latencies (P50, P95, P99) rather than averages, and benchmark pipeline vs. individual command patterns to quantify the gains. Test with realistic payload sizes and concurrency levels, and always run benchmarks in an environment that matches production to get meaningful comparisons.
