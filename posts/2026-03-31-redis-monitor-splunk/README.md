# How to Monitor Redis with Splunk

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Splunk, Monitoring, Observability, Log Management

Description: Learn how to monitor Redis with Splunk - covering the Splunk Add-on for Redis, log collection, metric ingestion, and building dashboards for Redis health monitoring.

---

Splunk provides Redis monitoring through the official Splunk Add-on for Redis, which collects metrics via the Redis INFO command, and through log collection for slow log and error analysis. This guide covers setup and practical monitoring use cases.

## Install the Splunk Add-on for Redis

Install the add-on from Splunkbase:

1. Go to Apps > Manage Apps > Browse More Apps
2. Search for "Splunk Add-on for Redis"
3. Install and configure

Or install via the CLI:

```bash
splunk install app splunk-add-on-for-redis.tgz -auth admin:password
```

## Configure the Redis Data Input

After installation, configure the data input:

```bash
# In Splunk Web: Settings > Data Inputs > Redis
# Or via inputs.conf

cat >> /opt/splunk/etc/apps/Splunk_TA_redis/local/inputs.conf << 'EOF'
[redis://default]
host = localhost
port = 6379
password =
interval = 60
index = redis_metrics
sourcetype = redis:info
EOF
```

Restart Splunk to apply:

```bash
splunk restart
```

## Collect Redis Slow Log

Configure the Universal Forwarder to collect Redis slow log entries:

```bash
# Enable slow log in redis.conf
# slowlog-log-slower-than 10000
# slowlog-max-len 128

# Export slow log to file using a cron job
cat > /etc/cron.d/redis-slowlog << 'EOF'
*/1 * * * * redis redis-cli SLOWLOG GET 50 >> /var/log/redis/slowlog.log
EOF
```

```bash
# inputs.conf on the Forwarder
[monitor:///var/log/redis/slowlog.log]
index = redis_logs
sourcetype = redis:slowlog
```

## Search Redis Metrics in Splunk

Basic SPL queries for Redis monitoring:

```text
# Memory usage over time
index=redis_metrics sourcetype=redis:info
| timechart avg(used_memory) as memory_bytes

# Cache hit rate
index=redis_metrics sourcetype=redis:info
| eval hit_rate = keyspace_hits / (keyspace_hits + keyspace_misses) * 100
| timechart avg(hit_rate) as cache_hit_rate_pct

# Connected clients
index=redis_metrics sourcetype=redis:info
| timechart max(connected_clients)
```

## Create a Redis Dashboard

Build a monitoring dashboard with these panels:

```text
Panel 1: Memory Usage
SPL: index=redis_metrics | timechart avg(used_memory_rss)

Panel 2: Cache Hit Rate
SPL: index=redis_metrics
  | eval hit_rate=keyspace_hits/(keyspace_hits+keyspace_misses)*100
  | timechart avg(hit_rate)

Panel 3: Commands Per Second
SPL: index=redis_metrics | timechart avg(instantaneous_ops_per_sec)

Panel 4: Slow Queries
SPL: index=redis_logs sourcetype=redis:slowlog
  | stats count by command
  | sort -count
```

## Set Up Alerts

Create a Splunk alert for high memory usage:

```text
Alert Name: Redis High Memory

Search:
index=redis_metrics sourcetype=redis:info
| eval mem_pct = used_memory / maxmemory * 100
| where mem_pct > 85

Schedule: Every 5 minutes
Alert Condition: Number of results > 0
Action: Send email / trigger webhook
```

## Forward Redis Logs for Error Analysis

Collect the main Redis log file for error pattern analysis:

```text
[monitor:///var/log/redis/redis-server.log]
index = redis_logs
sourcetype = redis:log

Search for errors:
index=redis_logs sourcetype=redis:log "MISCONF" OR "OOM" OR "READONLY"
| timechart count by log_level
```

## Summary

Splunk monitors Redis through the official add-on for structured metrics collection and the Universal Forwarder for log-based analysis. Use SPL to calculate cache hit rates, track memory growth, and identify slow commands. Alerts and dashboards built on Redis metrics give your operations team real-time visibility into Redis health without requiring additional infrastructure.
