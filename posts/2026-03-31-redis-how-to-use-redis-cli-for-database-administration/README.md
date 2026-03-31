# How to Use redis-cli for Database Administration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis-Cli, Database Administration, Operation, Monitoring

Description: Learn how to use redis-cli for common database administration tasks including monitoring, keyspace analysis, configuration management, and troubleshooting.

---

## Connecting to Redis with redis-cli

```bash
# Connect to local Redis
redis-cli

# Connect to remote Redis with authentication
redis-cli -h 192.168.1.100 -p 6379 -a your_password

# Connect with TLS
redis-cli -h redis.example.com -p 6380 --tls

# Connect to a specific database
redis-cli -n 3

# Run a single command and exit
redis-cli -h localhost PING
redis-cli -h localhost GET mykey
```

## Monitoring Real-Time Commands

MONITOR shows every command received by the server in real time:

```bash
redis-cli MONITOR
```

Example output:

```text
1711900000.123456 [0 127.0.0.1:54321] "SET" "user:1" "Alice"
1711900000.124000 [0 127.0.0.1:54321] "GET" "user:1"
1711900000.125000 [0 127.0.0.1:54321] "EXPIRE" "session:abc" "3600"
```

Use MONITOR sparingly - it has a performance cost. Pipe through grep to filter:

```bash
redis-cli MONITOR | grep -E "SET|DEL"
```

## Analyzing Slow Commands

The slow log records commands that exceed the configured threshold:

```bash
# View the last 10 slow commands
redis-cli SLOWLOG GET 10

# Reset the slow log
redis-cli SLOWLOG RESET

# Configure the threshold (microseconds)
redis-cli CONFIG SET slowlog-log-slower-than 10000  # 10ms
redis-cli CONFIG SET slowlog-max-len 256
```

Example slow log entry:

```text
1) 1) (integer) 14          # Log entry ID
   2) (integer) 1711900000  # Unix timestamp
   3) (integer) 25678       # Duration in microseconds
   4) 1) "KEYS"             # Command
      2) "*"
   5) "127.0.0.1:12345"     # Client address
   6) ""                    # Client name
```

## Keyspace Analysis and Scanning

Never use KEYS * in production - it blocks the server. Use SCAN instead:

```bash
# Scan keyspace incrementally (safe for production)
redis-cli SCAN 0 COUNT 100

# Scan with a pattern
redis-cli SCAN 0 MATCH "user:*" COUNT 100

# Iteratively scan all keys matching a pattern
redis-cli --scan --pattern "user:*"

# Count matching keys
redis-cli --scan --pattern "session:*" | wc -l
```

Analyze memory usage per key type:

```bash
# Check encoding and memory for a specific key
redis-cli OBJECT ENCODING mykey
redis-cli MEMORY USAGE mykey
redis-cli OBJECT IDLETIME mykey
```

## Server Information and Statistics

```bash
# Full server info
redis-cli INFO

# Specific sections
redis-cli INFO server
redis-cli INFO memory
redis-cli INFO clients
redis-cli INFO replication
redis-cli INFO stats
redis-cli INFO persistence
redis-cli INFO keyspace

# Combined: memory and clients
redis-cli INFO memory clients
```

Key metrics to monitor regularly:

```bash
redis-cli INFO memory | grep -E "used_memory_human|mem_fragmentation_ratio|maxmemory_human"
redis-cli INFO clients | grep -E "connected_clients|blocked_clients|tracking_clients"
redis-cli INFO stats | grep -E "total_commands_processed|instantaneous_ops_per_sec|rejected_connections"
redis-cli INFO replication | grep -E "role|connected_slaves|master_repl_offset"
```

## Configuration Management

```bash
# View all configuration
redis-cli CONFIG GET "*"

# View specific settings
redis-cli CONFIG GET maxmemory
redis-cli CONFIG GET maxmemory-policy
redis-cli CONFIG GET save

# Change configuration at runtime
redis-cli CONFIG SET maxmemory 4gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# Persist runtime changes to redis.conf
redis-cli CONFIG REWRITE
```

## Database Selection and Flushing

```bash
# Select a database (0-15)
redis-cli SELECT 3

# Get number of keys in current database
redis-cli DBSIZE

# Flush current database (careful!)
redis-cli FLUSHDB

# Flush all databases (very careful!)
redis-cli FLUSHALL

# Async flush (non-blocking)
redis-cli FLUSHDB ASYNC
```

## Backup and Restore via redis-cli

```bash
# Trigger RDB save
redis-cli BGSAVE

# Trigger AOF rewrite
redis-cli BGREWRITEAOF

# Check last save time
redis-cli LASTSAVE

# Load data from an RDB file (DEBUG only)
redis-cli DEBUG RELOAD
```

## Client Management

```bash
# List all connected clients
redis-cli CLIENT LIST

# Set a name for the current connection
redis-cli CLIENT SETNAME myapp-worker

# Kill a specific client
redis-cli CLIENT KILL ID 42

# Kill all clients from a specific address
redis-cli CLIENT KILL ADDR 192.168.1.50:54321

# Pause incoming client commands (for maintenance)
redis-cli CLIENT PAUSE 5000  # pause for 5 seconds
```

## Latency Monitoring

```bash
# Measure baseline latency
redis-cli --latency

# Measure latency over time
redis-cli --latency-history -i 5  # Sample every 5 seconds

# Get latency histogram
redis-cli --latency-dist

# Enable latency monitoring in server
redis-cli CONFIG SET latency-monitor-threshold 25  # Alert for > 25ms events
redis-cli LATENCY LATEST
redis-cli LATENCY HISTORY event_name
redis-cli LATENCY RESET
```

## Running Multiple Commands from a File

```bash
# Create a command file
cat > /tmp/redis-commands.txt << 'EOF'
PING
SET mykey hello
GET mykey
DBSIZE
EOF

# Execute all commands
redis-cli < /tmp/redis-commands.txt

# Execute in pipeline mode for speed
redis-cli --pipe < /tmp/redis-commands.txt
```

## Scripting with redis-cli

```bash
#!/bin/bash
# redis-admin-report.sh

REDIS_HOST="${1:-localhost}"
REDIS_PORT="${2:-6379}"
CLI="redis-cli -h $REDIS_HOST -p $REDIS_PORT"

echo "=== Redis Admin Report - $(date) ==="
echo ""
echo "--- Server Version ---"
$CLI INFO server | grep redis_version

echo ""
echo "--- Memory Usage ---"
$CLI INFO memory | grep -E "used_memory_human|used_memory_rss_human|mem_fragmentation_ratio|maxmemory_human"

echo ""
echo "--- Keyspace ---"
$CLI INFO keyspace

echo ""
echo "--- Replication ---"
$CLI INFO replication | grep -E "role|connected_slaves|master_link_status"

echo ""
echo "--- Slow Log (last 5) ---"
$CLI SLOWLOG GET 5

echo ""
echo "--- Recent Errors ---"
$CLI LATENCY LATEST
```

## Summary

redis-cli is an essential tool for Redis database administration, providing real-time monitoring via MONITOR, incremental keyspace scanning via SCAN, slow command analysis via SLOWLOG, and runtime configuration via CONFIG SET. Use INFO sections to monitor memory, clients, replication, and persistence metrics. Always prefer SCAN over KEYS for keyspace exploration in production, use CLIENT LIST to audit connections, and pipe redis-cli commands for bulk administration tasks.
