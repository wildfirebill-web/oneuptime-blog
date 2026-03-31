# How to Use mongostat to Monitor MongoDB Performance in Real Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongostat, Monitoring, Performance, Operations

Description: Learn how to use mongostat to monitor MongoDB server statistics in real time, including opcounters, memory usage, connections, and replication lag.

---

## What is mongostat

`mongostat` is a command-line tool that provides a rolling display of MongoDB server statistics, similar to the Unix `vmstat` command. It displays key metrics every second (by default) including operations per second, memory usage, connections, and replication status.

## Installation

mongostat is included in the MongoDB Database Tools package:

```bash
# On Ubuntu/Debian
sudo apt-get install mongodb-database-tools

# On macOS with Homebrew
brew install mongodb-database-tools

# Verify
mongostat --version
```

## Basic Usage

```bash
# Monitor local MongoDB every second
mongostat

# Monitor with a custom interval (every 5 seconds)
mongostat 5

# Monitor a remote server
mongostat --uri="mongodb://admin:password@db.example.com:27017" --authenticationDatabase=admin

# Monitor Atlas cluster
mongostat --uri="mongodb+srv://user:password@cluster0.example.mongodb.net"
```

## Understanding the Output

Sample output from mongostat:

```text
insert query update delete getmore command dirty used flushes vsize   res qrw arw net_in net_out conn                time
    *0    *0     *0     *0       0     1|0  0.0% 0.1%       0 1.54G 87.0M 0|0 1|0   111b    39.8k    5 Mar 31 02:00:01.234
     5    12      8      2       0    15|0  0.1% 0.2%       0 1.54G 87.0M 0|0 1|0   8.2k    12.5k    5 Mar 31 02:00:02.234
```

Key columns explained:
- `insert/query/update/delete`: Operations per second
- `getmore`: Cursor getMore operations per second
- `command`: Command operations per second
- `dirty`: Percentage of WiredTiger cache that is dirty
- `used`: Percentage of WiredTiger cache in use
- `flushes`: WiredTiger checkpoints per reporting interval
- `vsize`: Virtual memory size used by mongod
- `res`: Resident memory (actual RAM)
- `qrw`: Queued reads | queued writes
- `arw`: Active reads | active writes
- `net_in/net_out`: Network traffic in/out
- `conn`: Number of open connections

## Monitoring Multiple Hosts (Replica Set)

```bash
# Monitor all replica set members
mongostat   --host "rs0/mongo1:27017,mongo2:27017,mongo3:27017"   --username admin   --password secret   --authenticationDatabase admin

# Discover replica set members automatically
mongostat   --uri="mongodb://admin:secret@mongo1:27017/?replicaSet=rs0"
```

Output includes a `set` column showing the replica set member's role:

```text
                    host insert query update delete ... set repl                time
  mongo1:27017/rs0      5    12      3      1 ...  rs0  PRI Mar 31 02:00:01.234
  mongo2:27017/rs0     *0    *0     *0     *0 ...  rs0  SEC Mar 31 02:00:01.234
  mongo3:27017/rs0     *0    *0     *0     *0 ...  rs0  SEC Mar 31 02:00:01.234
```

## Filtering Output Fields

```bash
# Show only specific fields
mongostat --columns=insert,query,update,delete,conn,res

# Show all available fields
mongostat --all
```

## Limiting Output Duration

```bash
# Run for exactly 60 seconds (60 samples at 1/sec)
mongostat --rowcount=60

# Run for 30 samples at 2-second intervals
mongostat --rowcount=30 2
```

## Saving Output to a File

```bash
# Redirect output to a log file
mongostat --rowcount=3600 > /var/log/mongostat-$(date +%Y%m%d).log &

# JSON output for programmatic processing
mongostat --json --rowcount=10
```

JSON output example:

```json
{
  "localhost:27017": {
    "insert": 5,
    "query": 12,
    "update": 3,
    "delete": 1,
    "dirty": "0.1%",
    "used": "45.2%",
    "conn": 42,
    "res": "512M",
    "time": "2026-03-31T02:00:01Z"
  }
}
```

## Alerting Based on mongostat Output

```bash
#!/bin/bash
# monitor-connections.sh - Alert if connections exceed threshold

THRESHOLD=500
MONGO_URI="mongodb://admin:password@localhost:27017"

while true; do
  CONN=$(mongostat --uri="$MONGO_URI" --rowcount=1 --quiet --json     | python3 -c "import sys,json; data=json.load(sys.stdin); print(list(data.values())[0]['conn'])")

  if [ "$CONN" -gt "$THRESHOLD" ]; then
    echo "ALERT: MongoDB connections ($CONN) exceeded threshold ($THRESHOLD)"       | mail -s "MongoDB Alert" ops@example.com
  fi

  sleep 60
done
```

## Interpreting Common Issues

High `dirty` percentage (above 20%):
- Indicates WiredTiger cache is filling up faster than it can flush
- Consider increasing `wiredTigerCacheSizeGB` or reducing write load

High `qrw` (queued reads/writes):
- Operations are waiting for locks
- Investigate slow queries with `db.currentOp()` and the slow query log

Rapidly increasing `conn`:
- Connection leak in your application
- Review connection pool settings and ensure connections are properly released

## Summary

`mongostat` provides a real-time view of MongoDB server health with key metrics including opcounters, cache utilization, connections, and replication lag. It is essential for diagnosing performance problems, detecting anomalies, and understanding server load patterns. Use `--json` output for integration with monitoring pipelines and alerting systems.
