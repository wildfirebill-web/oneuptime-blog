# How to Audit Redis Access for Regulatory Compliance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Audit, Compliance, Security, ACL, Access Logging

Description: Audit Redis access for regulatory compliance using ACL logging, Redis MONITOR, keyspace notifications, and external log shipping to capture and retain access records.

---

## Compliance Requirements for Redis

Depending on your industry, regulations like SOC 2, PCI-DSS, HIPAA, and GDPR may require:
- Logging all read/write access to sensitive data
- Retaining access logs for 1-7 years
- Detecting unauthorized access attempts
- Demonstrating least-privilege access controls

## Step 1 - Enable ACL Logging

Redis 6+ includes an ACL log that records authorization failures:

```bash
# Configure ACL log max length (default: 128)
CONFIG SET acllog-max-len 1000

# View ACL access denials
ACL LOG

# Reset ACL log
ACL LOG RESET
```

Example output:

```text
1) 1) "count"
   2) (integer) 1
   3) "reason"
   4) "command"
   5) "object"
   6) "keys"
   7) "username"
   8) "app-service"
   9) "age-seconds"
   10) "0.127"
   11) "client-info"
   12) "..."
```

## Step 2 - Define Least-Privilege ACL Users

```bash
# Create read-only user for reporting service
ACL SETUSER reporting-svc on >secure_password ~data:* &* +@read

# Create write-only user for ingest service
ACL SETUSER ingest-svc on >secure_password ~ingest:* &* +SET +HSET +XADD

# Create admin user for operations
ACL SETUSER ops-admin on >admin_password ~* &* +@all

# View all users
ACL LIST
```

Save to file:

```bash
ACL SAVE
# Persists to the aclfile path in redis.conf
```

## Step 3 - Enable Keyspace Notifications

Keyspace notifications fire events on key operations, useful for detecting access to sensitive keys:

```bash
# Enable notifications for all key events (heavy - use targeted in production)
CONFIG SET notify-keyspace-events KEA

# Enable only for key access (reads) and expired events
CONFIG SET notify-keyspace-events Kxg
```

Subscribe to notifications in an audit consumer:

```python
import redis
import json
import logging

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(message)s")

r = redis.Redis(decode_responses=True)
r.config_set("notify-keyspace-events", "KEA")

pubsub = r.pubsub()
pubsub.psubscribe("__keyevent@0__:*")
pubsub.psubscribe("__keyspace@0__:*")

for message in pubsub.listen():
    if message["type"] in ("pmessage", "message"):
        logging.info(json.dumps({
            "channel": message.get("channel"),
            "pattern": message.get("pattern"),
            "data": message.get("data")
        }))
```

## Step 4 - Use MONITOR for Short-Term Audit Sessions

`MONITOR` streams every command processed by Redis in real-time. Use only for short forensic sessions:

```bash
redis-cli MONITOR | tee /tmp/redis-audit-$(date +%Y%m%d-%H%M).log
```

Stop after capturing a sample:

```bash
# Ctrl+C to stop
```

Parse the log for sensitive key access:

```bash
grep "GET\|SET\|HGET\|HSET" /tmp/redis-audit-*.log | grep "pii:"
```

## Step 5 - Ship Logs to Centralized Storage

```bash
# Install Fluent Bit or Filebeat to ship Redis logs
# Example Fluent Bit config snippet
cat > /etc/fluent-bit/redis-log.conf << 'EOF'
[INPUT]
    Name  tail
    Path  /var/log/redis/redis.log
    Tag   redis.access

[OUTPUT]
    Name  s3
    Match redis.*
    bucket my-audit-logs
    region us-east-1
    total_file_size 100M
    upload_timeout 10m
EOF
```

## Step 6 - Redis Slow Log for Detecting Unusual Activity

```bash
# Log commands slower than 1ms
CONFIG SET slowlog-log-slower-than 1000
CONFIG SET slowlog-max-len 500

# View slow log (may indicate table scans or expensive KEYS commands)
SLOWLOG GET 50
```

## Compliance Checklist

```text
[ ] ACL users created with least-privilege permissions
[ ] ACL SAVE configured to persist user definitions
[ ] ACL LOG enabled and monitored for denials
[ ] Keyspace notifications enabled for sensitive key patterns
[ ] Redis logs shipped to immutable audit log storage (S3 with Object Lock)
[ ] Log retention configured per compliance requirements (e.g. 1-7 years)
[ ] Regular ACL review schedule documented
[ ] Redis admin access restricted to named individuals
[ ] TLS enabled for data in transit
[ ] Passwords/auth strings rotated quarterly
```

## Python - Audit Report Generator

```python
import redis

r = redis.Redis(decode_responses=True)

def generate_acl_report():
    users = r.acl_list()
    print("=== Redis ACL User Report ===")
    for user in users:
        parts = user.split()
        username = parts[1]
        print(f"\nUser: {username}")
        info = r.acl_getuser(username)
        print(f"  Enabled: {info['flags']}")
        print(f"  Commands: {info['commands']}")
        print(f"  Keys: {info['keys']}")

    print("\n=== Recent ACL Denials ===")
    denials = r.acl_log()
    for denial in denials[:10]:
        print(f"  User: {denial['username']} | Reason: {denial['reason']} | Object: {denial['object']}")

generate_acl_report()
```

## Summary

Auditing Redis for regulatory compliance requires ACL-based least-privilege user definitions, ACL log monitoring for access denials, keyspace notifications for sensitive key access tracking, and shipping logs to immutable centralized storage. Regular ACL reviews and periodic log sampling with MONITOR complete the compliance posture.
