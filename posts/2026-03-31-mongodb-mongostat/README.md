# How to Use mongostat for Real-Time MongoDB Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Monitoring, Operations, Performance, Tools

Description: Learn how to use mongostat to view real-time MongoDB operation rates, connection counts, memory usage, and replication status from the command line.

---

## What is mongostat

`mongostat` is a command-line tool that displays a running summary of MongoDB's current activity at a configurable polling interval. It shows operation counts, connections, memory, and more - updated in real time. It is similar to the `iostat` or `vmstat` tools on Linux.

mongostat is part of the MongoDB Database Tools package, which is installed separately from the MongoDB server.

```mermaid
flowchart LR
    mongostat -->|serverStatus every N seconds| MongoDB
    MongoDB -->|opcounters, mem, connections| mongostat
    mongostat --> Terminal[Terminal output - refreshed every N seconds]
```

## Installation

On Ubuntu/Debian:

```bash
sudo apt-get install mongodb-database-tools
```

On RHEL/CentOS:

```bash
sudo yum install mongodb-database-tools
```

On macOS:

```bash
brew install mongodb/brew/mongodb-database-tools
```

## Basic Usage

Run mongostat against a local MongoDB instance with 1-second intervals (default):

```bash
mongostat --uri "mongodb://127.0.0.1:27017"
```

With authentication:

```bash
mongostat --uri "mongodb://adminUser:password@127.0.0.1:27017/?authSource=admin"
```

Set a 5-second interval:

```bash
mongostat --uri "mongodb://adminUser:password@127.0.0.1:27017/?authSource=admin" 5
```

Run for 60 seconds then exit (60 samples at 1s interval):

```bash
mongostat --uri "mongodb://adminUser:password@127.0.0.1:27017/?authSource=admin" --rowcount=60 1
```

## Reading the Output

A typical mongostat output line looks like this:

```text
insert query update delete getmore command dirty  used flushes vsize   res qrw arw net_in net_out conn                time
    *0    *0     *0     *0       0     1|0  0.0% 32.4%       0 1.45G 1.22G 0|0 0|0   111b    35.6k    3 Mar 31 10:00:00.000
   123   456     78     12       2    34|0  0.1% 32.5%       0 1.45G 1.22G 0|0 0|0   87.2k    234k    8 Mar 31 10:00:01.000
```

Column definitions:

| Column | Meaning |
|--------|---------|
| `insert` | Insert operations per polling interval |
| `query` | Query operations per interval |
| `update` | Update operations per interval |
| `delete` | Delete operations per interval |
| `getmore` | Getmore (cursor batch fetch) operations |
| `command` | Commands executed (format: `n\|repl`) |
| `dirty` | Percentage of WiredTiger cache with dirty (unwritten) pages |
| `used` | Percentage of WiredTiger cache currently in use |
| `flushes` | Number of WiredTiger checkpoints during the interval |
| `vsize` | Virtual memory size |
| `res` | Resident (physical) memory used |
| `qrw` | Queued reads and writes (r\|w) |
| `arw` | Active reads and writes (r\|w) |
| `net_in` | Bytes received over the network |
| `net_out` | Bytes sent over the network |
| `conn` | Number of open client connections |
| `time` | Timestamp of the sample |

## Connecting to a Replica Set

mongostat can display one line per replica set member when connected to the primary:

```bash
mongostat \
  --uri "mongodb://adminUser:password@primary:27017/?authSource=admin&replicaSet=rs0" \
  --discover
```

The `--discover` flag causes mongostat to discover and display metrics for all members of the replica set automatically.

## Monitoring a Sharded Cluster

Connect to a mongos router and use `--discover` to see all shard members:

```bash
mongostat \
  --uri "mongodb://adminUser:password@mongos:27017/?authSource=admin" \
  --discover
```

## Output Formats

Output as JSON for scripting or log aggregation:

```bash
mongostat \
  --uri "mongodb://adminUser:password@127.0.0.1:27017/?authSource=admin" \
  --json
```

Write output to a file while also displaying it:

```bash
mongostat \
  --uri "mongodb://adminUser:password@127.0.0.1:27017/?authSource=admin" \
  --rowcount=120 \
  1 | tee /tmp/mongostat-$(date +%Y%m%d-%H%M%S).log
```

## What to Watch For

**High `dirty` percentage (above 20%)**
WiredTiger is having trouble evicting dirty pages fast enough. This can indicate heavy write load or insufficient cache size.

**High `used` percentage (above 95%)**
The WiredTiger cache is nearly full. MongoDB will evict pages aggressively. Consider increasing `cacheSizeGB` or the server's RAM.

**High `qrw` values**
Reads and writes are queuing, indicating the server cannot keep up with the workload. Look for slow queries or missing indexes.

**Rising `conn` count**
A connection leak in the application - connections are being opened but not closed. Investigate the application's connection pool configuration.

**Low operation counts after a spike**
Can indicate a traffic drop or that slow queries are blocking other operations.

## Filtering Output Columns

Use `--columns` to show only specific fields:

```bash
mongostat \
  --uri "mongodb://adminUser:password@127.0.0.1:27017/?authSource=admin" \
  --columns "insert,query,update,delete,conn,used,dirty"
```

## Practical Example - Load Test Monitoring

During a load test, run mongostat in a separate terminal to watch live impact:

```bash
mongostat \
  --uri "mongodb://adminUser:password@127.0.0.1:27017/?authSource=admin" \
  --rowcount=300 \
  1 | tee loadtest-mongostat.log
```

After the test, analyze the log to find peak operation rates:

```bash
awk '{print $1, $2, $3, $4, $NF}' loadtest-mongostat.log | sort -k1 -n | tail -20
```

## Best Practices

- Run mongostat from a monitoring host, not the MongoDB server, to avoid adding CPU load to the database.
- Use `--rowcount` to avoid running indefinitely in automated scripts.
- For persistent monitoring, use Prometheus with the MongoDB exporter rather than mongostat.
- Use mongostat for interactive, real-time investigation during incidents or load tests.
- Check `dirty` and `used` percentages together - high `dirty` with high `used` is a sign of memory pressure.

## Summary

mongostat provides a live, terminal-friendly view of MongoDB's operation rates, memory usage, and connection counts. Use it during load tests and incident response to quickly understand what MongoDB is doing right now. The most important columns to watch are `insert/query/update/delete` for operation rates, `conn` for connection counts, `dirty` and `used` for WiredTiger cache health, and `qrw` for queue buildup. For continuous monitoring in production, use Prometheus and Grafana alongside mongostat.
