# How to Interpret Redis Benchmark Results

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Benchmark, Performance, Throughput, Latency

Description: Learn how to read and interpret redis-benchmark output, understand throughput and latency percentiles, and identify what the numbers actually mean for production.

---

Running `redis-benchmark` is easy, but interpreting its output correctly is where most people struggle. This guide walks through each section of benchmark output and explains what to look for.

## Run a Basic Benchmark

```bash
redis-benchmark -h 127.0.0.1 -p 6379 -n 100000 -c 50 -t set,get
```

Parameters:
- `-n 100000`: total number of requests
- `-c 50`: 50 concurrent clients
- `-t set,get`: only test SET and GET

## Reading the Output

```text
====== SET ======
  100000 requests completed in 1.23 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1
  host configuration "save": ""
  host configuration "appendonly": "no"
  multi-thread: no

99.56% <= 1 milliseconds
99.90% <= 2 milliseconds
100.00% <= 3 milliseconds
81300.81 requests per second, p50=0.487 ms (RPS=81300.81, P50=0.487, P99=1.471, P99.9=2.271)
```

### Throughput (RPS)

`81300.81 requests per second` is your throughput. This represents how many operations Redis can handle per second under the test conditions. Higher is better, but always compare it to your actual production load - if you need 10K ops/sec, 81K is well above requirement.

### Latency Percentiles

| Percentile | Meaning |
|---|---|
| p50 | Half of all requests finished in under this time |
| p99 | 99% of requests finished in under this time |
| p99.9 | 99.9% of requests finished in under this time |

P99 and P99.9 are most important for user-facing applications. A p50 of 0.5ms is excellent, but a p99.9 of 10ms means 1 in 1000 requests will feel slow.

## Compare Different Configurations

Test with and without pipelining:

```bash
# Without pipelining
redis-benchmark -n 100000 -c 50 -t set -q

# With pipelining (16 commands per pipeline)
redis-benchmark -n 100000 -c 50 -t set -P 16 -q
```

```text
Without: SET: 82000 requests/sec
With P16: SET: 620000 requests/sec
```

Pipelining dramatically improves throughput at the cost of per-command latency visibility.

## Test with Realistic Payload Sizes

Default payload is 3 bytes. Use realistic sizes:

```bash
# 1KB payload (typical cache value)
redis-benchmark -n 50000 -c 50 -t set -d 1024 -q

# 10KB payload
redis-benchmark -n 10000 -c 50 -t set -d 10240 -q
```

Larger payloads reduce throughput significantly.

## Warning Signs in Results

- P99 > 10ms: high tail latency, investigate slow commands or resource contention
- P99.9 >> P99: occasional extreme outliers, often caused by `fork()` for RDB saves
- RPS much lower than baseline: check CPU, network, or persistence settings

## Summary

Benchmark results report throughput in requests/sec and latency at various percentiles. Focus on P99 and P99.9 for user-facing impact - high P50 throughput means nothing if P99 latency is unacceptable. Always test with realistic concurrency levels, payload sizes, and the same persistence settings as production.
