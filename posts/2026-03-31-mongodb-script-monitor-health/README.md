# How to Write a Script to Monitor MongoDB Health

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Scripting, Monitoring, Operation, Shell

Description: Learn how to write a shell and Python script to monitor MongoDB health by checking server status, replication lag, connection counts, and slow operation alerts.

---

## Overview

A MongoDB health monitoring script gives you a quick operational snapshot: is the server up, is replication healthy, are connections near the limit, and are there slow queries. This guide builds a script that checks all of these and outputs a structured health report.

## Shell Health Check Script

```bash
#!/bin/bash
# mongodb-health-check.sh

MONGO_URI="${MONGO_URI:-mongodb://localhost:27017}"
ALERT_EMAIL="ops-team@example.com"
REPL_LAG_THRESHOLD=30  # seconds
CONN_USAGE_THRESHOLD=80  # percent

echo "=== MongoDB Health Check: $(date -u) ==="

# 1. Check if mongod is reachable
STATUS=$(mongosh "$MONGO_URI" --quiet --eval "db.runCommand({ ping: 1 }).ok" 2>&1)
if [ "$STATUS" != "1" ]; then
  echo "CRITICAL: MongoDB is not reachable!"
  exit 1
fi
echo "OK: MongoDB is reachable"

# 2. Check replication lag
REPL_LAG=$(mongosh "$MONGO_URI" --quiet --eval "
  const status = rs.status();
  const primary = status.members.find(m => m.stateStr === 'PRIMARY');
  const secondary = status.members.find(m => m.stateStr === 'SECONDARY');
  if (!primary || !secondary) { print(0); }
  else { print(Math.abs(primary.optimeDate - secondary.optimeDate) / 1000); }
" 2>/dev/null)

if [ -n "$REPL_LAG" ] && [ "$(echo "$REPL_LAG > $REPL_LAG_THRESHOLD" | bc)" = "1" ]; then
  echo "WARNING: Replication lag is ${REPL_LAG}s (threshold: ${REPL_LAG_THRESHOLD}s)"
else
  echo "OK: Replication lag is ${REPL_LAG:-N/A}s"
fi

# 3. Check connection usage
mongosh "$MONGO_URI" --quiet --eval "
  const ss = db.serverStatus();
  const current = ss.connections.current;
  const available = ss.connections.available;
  const total = current + available;
  const pct = Math.round((current / total) * 100);
  print('Connections: ' + current + '/' + total + ' (' + pct + '%)');
  if (pct > $CONN_USAGE_THRESHOLD) { print('WARNING: Connection usage is high'); }
  else { print('OK: Connection usage is normal'); }
"

# 4. Check for long-running operations
mongosh "$MONGO_URI" --quiet --eval "
  const ops = db.currentOp({ active: true, secs_running: { \$gt: 5 } });
  if (ops.inprog.length > 0) {
    print('WARNING: ' + ops.inprog.length + ' operations running > 5s');
    ops.inprog.forEach(op => print('  opid=' + op.opid + ' secs=' + op.secs_running + ' ns=' + op.ns));
  } else {
    print('OK: No long-running operations');
  }
"

echo "=== Health Check Complete ==="
```

## Python Health Check Script

```python
#!/usr/bin/env python3
import os
from datetime import datetime
from pymongo import MongoClient

MONGO_URI = os.environ.get("MONGO_URI", "mongodb://localhost:27017")
client = MongoClient(MONGO_URI, serverSelectionTimeoutMS=5000)

def check_health():
    report = {"timestamp": datetime.utcnow().isoformat(), "checks": []}

    # Ping
    try:
        client.admin.command("ping")
        report["checks"].append({"name": "connectivity", "status": "OK"})
    except Exception as e:
        report["checks"].append({"name": "connectivity", "status": "CRITICAL", "detail": str(e)})
        return report

    # Server status
    ss = client.admin.command("serverStatus")
    connections = ss["connections"]
    total = connections["current"] + connections["available"]
    conn_pct = round((connections["current"] / total) * 100)
    status = "WARNING" if conn_pct > 80 else "OK"
    report["checks"].append({
        "name": "connections",
        "status": status,
        "detail": f"{connections['current']}/{total} ({conn_pct}%)"
    })

    # Replication
    try:
        rs = client.admin.command("replSetGetStatus")
        members = rs.get("members", [])
        primary = next((m for m in members if m["stateStr"] == "PRIMARY"), None)
        secondaries = [m for m in members if m["stateStr"] == "SECONDARY"]
        if primary and secondaries:
            lag = abs((primary["optimeDate"] - secondaries[0]["optimeDate"]).total_seconds())
            status = "WARNING" if lag > 30 else "OK"
            report["checks"].append({"name": "replication_lag", "status": status, "detail": f"{lag:.1f}s"})
    except Exception:
        report["checks"].append({"name": "replication_lag", "status": "N/A", "detail": "Not a replica set"})

    return report

if __name__ == "__main__":
    report = check_health()
    print(f"Health Check: {report['timestamp']}")
    for check in report["checks"]:
        print(f"  [{check['status']}] {check['name']}: {check.get('detail', '')}")
```

## Scheduling the Script

```bash
# Run health check every 5 minutes via cron
*/5 * * * * /usr/local/bin/python3 /opt/scripts/mongodb_health.py >> /var/log/mongodb-health.log 2>&1
```

## Best Practices

- Use `serverSelectionTimeoutMS=5000` to avoid scripts hanging when MongoDB is unreachable.
- Check `ss.opcounters` from `serverStatus` to detect query storms (sudden spike in queries per second).
- Pipe output to a monitoring platform webhook (PagerDuty, OneUptime) rather than just logging to a file.

## Summary

A MongoDB health check script should verify connectivity, check replication lag, monitor connection usage, and detect long-running operations. Run it via cron every few minutes and alert when any check exceeds its threshold.
