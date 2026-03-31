# How to Plan Redis CPU Requirements Based on Command Mix

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Performance, Capacity Planning

Description: Learn how to analyze your Redis command mix to accurately estimate CPU requirements and avoid performance bottlenecks in production.

---

Redis is single-threaded for command processing, which means CPU planning requires understanding which commands your application actually uses - not just total operations per second.

## Measure Your Current Command Mix

Use `INFO commandstats` to see exactly which commands are being used and how much CPU they consume:

```bash
redis-cli INFO commandstats | sort -t= -k2 -rn | head -20
```

This outputs each command with call count, total CPU time, and microseconds per call:

```text
cmdstat_get:calls=9823411,usec=4912345,usec_per_call=0.50
cmdstat_set:calls=1234567,usec=2345678,usec_per_call=1.90
cmdstat_zadd:calls=345678,usec=3456780,usec_per_call=10.00
cmdstat_zrangebyscore:calls=123456,usec=6172800,usec_per_call=50.00
```

## Understand CPU Cost by Command Category

Not all commands are equal. O(1) commands like `GET`, `SET`, `HGET` are cheap. O(N) or O(N log N) commands like `SORT`, `ZUNIONSTORE`, or `SMEMBERS` on large sets are expensive.

```python
# Categorize commands by complexity
command_costs = {
    "O(1)": ["GET", "SET", "HGET", "HSET", "LPUSH", "RPUSH"],
    "O(log N)": ["ZADD", "ZRANK", "ZINCRBY"],
    "O(N)": ["SMEMBERS", "LRANGE", "KEYS", "HGETALL"],
    "O(N log N)": ["SORT", "ZUNIONSTORE", "ZINTERSTORE"],
}

def estimate_cpu_weight(command, count):
    weights = {"O(1)": 1, "O(log N)": 3, "O(N)": 10, "O(N log N)": 30}
    for complexity, cmds in command_costs.items():
        if command.upper() in cmds:
            return count * weights[complexity]
    return count * 2  # default moderate cost
```

## Run a CPU Profiling Baseline

Capture command stats over a representative time window:

```bash
# Snapshot before
redis-cli INFO commandstats > /tmp/before.txt

# Wait 60 seconds under load
sleep 60

# Snapshot after
redis-cli INFO commandstats > /tmp/after.txt

# Diff to see commands in that window
diff /tmp/before.txt /tmp/after.txt
```

## Estimate CPU Cores Needed

Redis I/O threads (Redis 6+) handle network separately, but command processing stays on one thread. A single Redis thread can handle roughly 50,000-100,000 simple commands per second on modern hardware.

```bash
# Check current ops/sec
redis-cli INFO stats | grep instantaneous_ops_per_sec

# Check how much CPU Redis is actually using
top -p $(pgrep redis-server) -bn1 | grep redis-server
```

If your workload has many O(N) commands, reduce the effective throughput estimate. A server with 20% heavy commands should be planned at 40-60% of the simple-command throughput ceiling.

## Plan Headroom and Scaling Triggers

```bash
# Monitor CPU util continuously
watch -n 5 'redis-cli INFO stats | grep -E "instantaneous_ops|used_cpu"'
```

Set up alerts when Redis CPU exceeds 70% sustained. At that point, consider:
- Moving expensive commands to replicas
- Using Redis Cluster to distribute load
- Pre-computing expensive aggregations

## Summary

Planning Redis CPU requires profiling your actual command mix rather than relying on generic benchmarks. Use `INFO commandstats` to identify expensive commands, estimate weighted CPU cost, and set scaling triggers before you hit the single-thread ceiling. Monitoring with OneUptime helps you track CPU trends over time and alert before saturation occurs.
