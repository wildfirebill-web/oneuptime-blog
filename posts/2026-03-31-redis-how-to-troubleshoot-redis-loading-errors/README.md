# How to Troubleshoot Redis LOADING Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Troubleshooting, LOADING, RDB, AOF, Startup

Description: Fix Redis LOADING errors that occur when Redis is still loading its dataset from RDB or AOF persistence files after a restart.

---

## What is the LOADING Error

When Redis restarts after enabling persistence (RDB snapshots or AOF), it reads the dataset from disk before accepting client commands. During this loading phase every command returns:

```text
LOADING Redis is loading the dataset in memory
```

This is normal behaviour, not a bug. The error resolves automatically once loading completes. The concern is when loading takes longer than expected or appears stuck.

## Step 1 - Check Loading Progress

```bash
redis-cli INFO persistence | grep -E 'loading|rdb|aof'
```

Example output during loading:

```text
loading:1
loading_start_time:1711900000
loading_loaded_bytes:536870912
loading_total_bytes:2147483648
loading_loaded_perc:25.00
loading_eta_seconds:45
```

`loading:0` means Redis has finished loading. `loading_eta_seconds` gives an estimate of remaining time.

## Step 2 - Estimate Normal Loading Time

Loading speed depends on dataset size, disk I/O speed, and whether compression is enabled.

Rough benchmarks:

- NVMe SSD: 1-2 GB/s throughput - a 10 GB RDB file loads in 5-15 seconds
- SATA SSD: ~500 MB/s - 10 GB in 20-30 seconds
- HDD: ~100 MB/s - 10 GB in 100+ seconds

If loading is slower than expected, check disk I/O:

```bash
iostat -x 1 5
```

## Step 3 - Check for Loading Errors in Logs

```bash
tail -200 /var/log/redis/redis-server.log | grep -iE 'error|corrupt|loading|rdb|aof'
```

If the RDB or AOF file is corrupted, Redis will log the error and may exit or start with an empty dataset depending on configuration.

## Step 4 - Validate the RDB File

```bash
redis-check-rdb /var/lib/redis/dump.rdb
```

A healthy file returns:

```text
[offset 0] Checking RDB file dump.rdb
[offset 26] AUX FIELD redis-ver = '7.0.11'
...
[chksum   ] Checksum OK
```

If the checksum fails:

```text
[chksum   ] Checksum ERROR!
```

You can force Redis to load a corrupted RDB by setting `rdbchecksum no` in `redis.conf`, but data loss may occur.

## Step 5 - Validate the AOF File

```bash
redis-check-aof /var/lib/redis/appendonly.aof
```

To auto-repair a truncated AOF:

```bash
redis-check-aof --fix /var/lib/redis/appendonly.aof
```

This truncates the file at the last valid command. Some data at the tail may be lost.

## Step 6 - Configure Clients to Wait During Loading

Some Redis clients allow automatic retry during the LOADING period. In redis-py:

```python
import redis
from redis.retry import Retry
from redis.backoff import ExponentialBackoff

r = redis.Redis(
    host='localhost',
    port=6379,
    retry=Retry(ExponentialBackoff(), retries=10),
    retry_on_error=[redis.exceptions.BusyLoadingError]
)
```

In Node.js with ioredis:

```javascript
const Redis = require('ioredis');

const client = new Redis({
  host: 'localhost',
  port: 6379,
  maxRetriesPerRequest: 20,
  retryStrategy(times) {
    return Math.min(times * 100, 3000);
  }
});
```

## Step 7 - Reduce Loading Time for Future Restarts

Options to reduce dataset loading time:

1. Enable RDB compression (default is `yes` - keep it enabled):

```text
rdbcompression yes
```

2. Disable AOF if you only need RDB:

```text
appendonly no
```

3. Use AOF with RDB preamble for faster loading:

```text
aof-use-rdb-preamble yes
```

4. Place the data directory on fast storage (NVMe SSD).

5. Consider Redis replication: a replica can serve reads while the primary loads.

## Summary

The Redis LOADING error is a normal transient state while Redis loads its persistence files on startup. Monitor progress with `INFO persistence`, validate RDB and AOF files with `redis-check-rdb` and `redis-check-aof` if loading fails, and configure your clients to retry during the loading phase. Use `aof-use-rdb-preamble yes` and fast storage to minimize loading time for large datasets.
