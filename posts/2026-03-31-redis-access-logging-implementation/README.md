# How to Implement Redis Access Logging

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Security, Logging, Monitoring, ACL

Description: Learn how to implement Redis access logging using ACL LOG, the MONITOR command, and external tools to track who is accessing what data.

---

Redis does not have comprehensive access logging built in, but combining ACL LOG, MONITOR, and log-level settings gives you visibility into access patterns, failed authentications, and denied commands.

## Method 1: ACL LOG for Denied Access

`ACL LOG` captures authentication failures and permission denials - the most security-relevant events:

```bash
# View recent ACL violations
ACL LOG

# View last 20 entries
ACL LOG 20
```

Sample output:

```text
1) 1) "count"
   2) (integer) 3
   3) "reason"
   4) "command"
   5) "context"
   6) "toplevel"
   7) "object"
   8) "flushall"
   9) "username"
   10) "appuser"
   11) "age-seconds"
   12) "0.542"
   13) "client-info"
   14) "id=4 addr=10.0.1.10:52341..."
```

Configure the ACL log size:

```text
# /etc/redis/redis.conf
acllog-max-len 256
```

Clear the log periodically:

```bash
ACL LOG RESET
```

## Method 2: Enable Verbose Connection Logging

Log every client connection and disconnection:

```text
# /etc/redis/redis.conf
loglevel verbose
logfile /var/log/redis/redis-server.log
```

This logs connection accepted and closed events with client IP and port.

## Method 3: MONITOR for Real-Time Command Capture

`MONITOR` streams every command executed in real time - use this for debugging, not permanent logging (it has performance impact):

```bash
redis-cli MONITOR
```

Sample output:

```text
1711234567.123456 [0 10.0.1.10:52341] "GET" "user:123"
1711234567.234567 [0 10.0.1.11:52342] "SET" "session:abc" "data"
```

Capture to a file for analysis:

```bash
redis-cli MONITOR > /tmp/redis-monitor.log &
# Run your test workload
kill %1
```

## Method 4: Parse Redis Logs for Access Patterns

```bash
# Count connections per source IP
grep "Accepted" /var/log/redis/redis-server.log | \
  grep -oP '\d+\.\d+\.\d+\.\d+' | sort | uniq -c | sort -rn

# Find authentication errors
grep -i "auth\|wrong\|denied" /var/log/redis/redis-server.log
```

## Method 5: Ship Logs to a Centralized System

Forward Redis logs to your SIEM or log aggregator:

```bash
# Filebeat config for Redis logs
cat > /etc/filebeat/inputs.d/redis.yml << 'EOF'
- type: log
  paths:
    - /var/log/redis/redis-server.log
  fields:
    service: redis
    environment: production
EOF
```

## Summary

Redis access logging combines three tools: `ACL LOG` captures authentication failures and permission denials, verbose `loglevel` logs connection events, and `MONITOR` streams all commands in real time. For production, use `ACL LOG` for security events and ship Redis logs to a centralized SIEM. Avoid running `MONITOR` continuously as it impacts throughput.
