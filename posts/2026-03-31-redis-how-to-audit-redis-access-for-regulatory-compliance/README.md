# How to Audit Redis Access for Regulatory Compliance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Security, Compliance, Auditing, Access Control

Description: Learn how to implement Redis access auditing for regulatory compliance using ACL logs, keyspace notifications, and log forwarding to SIEM systems.

---

## Compliance Requirements for Redis

Organizations subject to PCI DSS, HIPAA, SOC 2, or GDPR often need to:

- Log all privileged data access
- Record configuration changes
- Maintain audit trails for authentication events
- Alert on unauthorized access attempts
- Demonstrate access control enforcement

Redis provides ACL access logs, command-level logging, and keyspace notifications to meet these requirements.

## Enabling ACL Logging

Redis ACL logging records authorization failures when clients attempt to use commands or access keys they are not permitted to use.

Configure ACL log size in redis.conf:

```text
acllog-max-len 128
```

Or at runtime:

```bash
redis-cli CONFIG SET acllog-max-len 256
```

View the ACL log:

```bash
redis-cli ACL LOG
```

Example output:

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
   10) "readonly-user"
   11) "age-seconds"
   12) "120.5"
   13) "client-info"
   14) "id=42 addr=10.0.1.5:54321 cmd=flushall"
```

Reset the ACL log:

```bash
redis-cli ACL LOG RESET
```

## Setting Up Least-Privilege Users with ACL

For compliance, define users with minimum required permissions:

```bash
# Create a read-only user for application reporting
redis-cli ACL SETUSER readonly-reporter on >strong-password123 \
  ~report:* \
  ~analytics:* \
  +GET +HGET +HMGET +HGETALL +SMEMBERS +LRANGE +ZRANGE +ZREVRANGE

# Create a write user scoped to specific key patterns
redis-cli ACL SETUSER app-writer on >strong-password456 \
  ~app:* \
  +SET +SETEX +SETNX +HSET +LPUSH +ZADD +DEL +EXPIRE \
  -FLUSHDB -FLUSHALL -DEBUG -CONFIG -KEYS

# Create an admin user with full access but logged
redis-cli ACL SETUSER admin-user on >admin-password789 \
  ~* \
  +@all

# List all users
redis-cli ACL LIST
```

Persist ACL rules to a file:

```text
# In redis.conf
aclfile /etc/redis/users.acl
```

```bash
redis-cli ACL SAVE
```

## Logging All Commands with Slow Log

For compliance, configure slow log to capture commands above a threshold:

```bash
# Log all commands (threshold = 0)
redis-cli CONFIG SET slowlog-log-slower-than 0
redis-cli CONFIG SET slowlog-max-len 1024

# View slow log entries
redis-cli SLOWLOG GET 100

# Format: [id, timestamp, duration_us, [command, args], client_addr, client_name]
```

## Using MONITOR for Real-Time Access Logging

For short-term compliance audits, MONITOR captures every command:

```bash
# Capture all commands to a file for 60 seconds
timeout 60 redis-cli MONITOR > /var/log/redis/access-$(date +%Y%m%d%H%M%S).log
```

Parse the log for sensitive command patterns:

```bash
# Find GET commands accessing credit card patterns
grep -E '"GET" "cc:[0-9]' /var/log/redis/access-*.log

# Find FLUSHDB/FLUSHALL attempts
grep -E '"FLUSH' /var/log/redis/access-*.log
```

Note: MONITOR has performance impact (up to 50% throughput reduction). Use only for targeted audits.

## Keyspace Notifications for Data Access Auditing

Keyspace notifications publish messages when specific key events occur:

```bash
# Enable notifications for all commands on all keys
redis-cli CONFIG SET notify-keyspace-events "KEA"

# Or specific events:
# K = Keyspace events (per key)
# E = Keyevent events (per event type)
# g = Generic: DEL, EXPIRE, RENAME
# $ = String commands
# l = List commands
# s = Set commands
# h = Hash commands
# z = Sorted set commands
# x = Expired events
# d = Module key type events
redis-cli CONFIG SET notify-keyspace-events "KEx"
```

Subscribe to audit events in Python:

```python
import redis
import json
import logging
from datetime import datetime

logger = logging.getLogger('redis-audit')

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
pubsub = r.pubsub()

# Subscribe to all keyevent notifications
pubsub.psubscribe('__keyevent@0__:*')

def audit_handler(message):
    if message['type'] == 'pmessage':
        event_type = message['channel'].split(':')[1]
        key = message['data']
        audit_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'event': event_type,
            'key': key,
            'database': 0
        }
        logger.info(json.dumps(audit_entry))
        print(json.dumps(audit_entry))

for message in pubsub.listen():
    audit_handler(message)
```

## Forwarding Audit Logs to SIEM

Ship Redis logs to Elasticsearch, Splunk, or your SIEM:

```python
import redis
import json
import requests
from datetime import datetime
import threading

class RedisAuditLogger:
    def __init__(self, redis_host: str, siem_endpoint: str):
        self.r = redis.Redis(host=redis_host, port=6379, decode_responses=True)
        self.siem_endpoint = siem_endpoint
        self.pubsub = self.r.pubsub()

    def start(self):
        self.pubsub.psubscribe('__keyevent@0__:*')
        thread = threading.Thread(target=self._listen, daemon=True)
        thread.start()
        return thread

    def _listen(self):
        for message in self.pubsub.listen():
            if message['type'] == 'pmessage':
                self._send_to_siem(message)

    def _send_to_siem(self, message):
        event = {
            'timestamp': datetime.utcnow().isoformat() + 'Z',
            'source': 'redis',
            'event_type': message['channel'].split(':')[1],
            'key': message['data'],
            'severity': 'info'
        }
        try:
            requests.post(
                self.siem_endpoint,
                json=event,
                timeout=5,
                headers={'Content-Type': 'application/json'}
            )
        except Exception as e:
            print(f"Failed to send audit event: {e}")

# Usage
logger = RedisAuditLogger(
    redis_host='localhost',
    siem_endpoint='https://your-siem-endpoint.com/ingest'
)
logger.start()
```

## Compliance Report: ACL and Access Summary

```python
import redis
import json
from datetime import datetime

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def generate_compliance_report():
    report = {
        'generated_at': datetime.utcnow().isoformat(),
        'users': [],
        'acl_violations': [],
        'server_info': {}
    }

    # List all users and their permissions
    for user_info in r.acl_list():
        report['users'].append(user_info)

    # Get ACL log violations
    violations = r.acl_log()
    for v in violations:
        report['acl_violations'].append({
            'username': v.get('username'),
            'reason': v.get('reason'),
            'object': v.get('object'),
            'count': v.get('count')
        })

    # Server security info
    info = r.info()
    report['server_info'] = {
        'redis_version': info.get('redis_version'),
        'connected_clients': info.get('connected_clients'),
        'total_commands_processed': info.get('total_commands_processed')
    }

    return json.dumps(report, indent=2)

print(generate_compliance_report())
```

## Summary

Auditing Redis access for regulatory compliance combines ACL logging for authorization failures, ACL user rules for least-privilege access control, keyspace notifications for data access events, and slow log configuration for command-level recording. Forward audit events to your SIEM using keyspace notification subscribers or periodic log exports. Persist ACL rules to an aclfile for reproducible configuration, review ACL LOG regularly for unauthorized access attempts, and generate periodic compliance reports from ACL LIST and server INFO data.
