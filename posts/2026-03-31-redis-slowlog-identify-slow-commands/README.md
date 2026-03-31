# How to Use SLOWLOG in Redis to Identify Slow Commands

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, SLOWLOG, Performance, Debugging, Latency

Description: Learn how to use Redis SLOWLOG to capture and analyze commands that exceed a latency threshold, helping you identify and fix performance bottlenecks.

---

## What Is SLOWLOG?

Redis `SLOWLOG` records commands that take longer than a configurable threshold to execute. Unlike application-level profiling, it captures execution time on the Redis server itself - giving you accurate server-side latency measurements.

## Configuring SLOWLOG

Two configuration parameters control SLOWLOG behavior:

- `slowlog-log-slower-than` - threshold in microseconds (default: 10000 = 10ms)
- `slowlog-max-len` - maximum number of entries to keep (default: 128)

```bash
# Log commands taking more than 5ms
redis-cli CONFIG SET slowlog-log-slower-than 5000

# Keep up to 256 entries
redis-cli CONFIG SET slowlog-max-len 256
```

To disable SLOWLOG:

```bash
redis-cli CONFIG SET slowlog-log-slower-than -1
```

## Viewing Slow Log Entries

```bash
redis-cli SLOWLOG GET
```

To get a specific number of entries:

```bash
redis-cli SLOWLOG GET 10
```

Example output:

```text
1) 1) (integer) 42          # Entry ID
   2) (integer) 1711872000  # Unix timestamp
   3) (integer) 15234       # Execution time in microseconds
   4) 1) "KEYS"             # Command
      2) "*"
   5) "127.0.0.1:52301"     # Client address
   6) ""                    # Client name
```

## Checking the Log Length

```bash
redis-cli SLOWLOG LEN
```

Returns the number of entries currently in the slow log.

## Resetting the Slow Log

```bash
redis-cli SLOWLOG RESET
```

Clears all entries from the slow log. Useful before running a benchmark or starting a new investigation window.

## Common Offenders in SLOWLOG

Commands that frequently appear in slow logs:

| Command | Reason |
|---------|--------|
| `KEYS *` | Full keyspace scan - O(N) |
| `SMEMBERS` on large sets | Returns all members |
| `LRANGE 0 -1` | Returns full list |
| `SORT` without LIMIT | O(N+M*log(M)) |
| `HGETALL` on large hashes | Serializes entire hash |

## Automating SLOWLOG Monitoring

```bash
#!/bin/bash
COUNT=$(redis-cli SLOWLOG LEN)
if [ "$COUNT" -gt 50 ]; then
  echo "WARNING: Redis slow log has $COUNT entries"
  redis-cli SLOWLOG GET 10
fi
```

## Exporting SLOWLOG to JSON with Python

```python
import redis, json

r = redis.Redis(host='localhost', port=6379)
entries = r.slowlog_get(20)
for entry in entries:
    print(json.dumps({
        'id': entry['id'],
        'duration_us': entry['duration'],
        'command': entry['command'].decode()
    }))
```

## Summary

`SLOWLOG` is Redis's built-in latency profiler. By capturing commands that exceed your threshold, it helps you identify O(N) commands, large data structure scans, and other performance issues before they impact users. Review your slow log regularly and adjust thresholds based on your latency requirements.
