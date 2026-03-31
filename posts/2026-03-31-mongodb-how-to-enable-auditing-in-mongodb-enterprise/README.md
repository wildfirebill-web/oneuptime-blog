# How to Enable Auditing in MongoDB Enterprise

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Auditing, Security, Compliance, Enterprise

Description: Enable MongoDB Enterprise auditing to record authentication attempts, CRUD operations, and schema changes for security compliance and forensics.

---

## What MongoDB Auditing Captures

MongoDB Enterprise auditing logs database operations to a file or syslog. Audit events include:
- Authentication (success and failure)
- Authorization failures
- Schema operations (create/drop collections, create/drop indexes)
- User and role management operations
- CRUD operations (configurable)

## Enabling Auditing in mongod.conf

```yaml
# /etc/mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/auditLog.json
```

Format options: `JSON` (structured, machine-readable) or `BSON` (compact binary).

Restart mongod to activate:

```bash
sudo systemctl restart mongod
```

## Filtering Audit Events

Without filtering, auditing generates enormous log volume. Use the `filter` option to audit only relevant events.

Audit only authentication and authorization events:

```yaml
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/auditLog.json
  filter: '{
    atype: {
      $in: [
        "authenticate",
        "authCheck",
        "logout",
        "createUser",
        "dropUser",
        "createRole",
        "dropRole"
      ]
    }
  }'
```

Audit all CRUD operations on a specific collection:

```yaml
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/auditLog.json
  filter: '{
    atype: "authCheck",
    "param.ns": "myapp.payments"
  }'
```

## Audit Log Entry Format

A typical audit log entry looks like:

```json
{
  "atype": "authenticate",
  "ts": { "$date": "2026-03-31T10:00:00.000Z" },
  "uuid": { "$binary": "..." },
  "local": { "ip": "127.0.0.1", "port": 27017 },
  "remote": { "ip": "10.0.0.5", "port": 54321 },
  "users": [{ "user": "appUser", "db": "myapp" }],
  "roles": [{ "role": "readWrite", "db": "myapp" }],
  "param": { "user": "appUser", "db": "myapp", "mechanism": "SCRAM-SHA-256" },
  "result": 0
}
```

`result: 0` means success. Non-zero values indicate errors.

## Parsing Audit Logs

Parse JSON audit logs with standard tools:

```bash
# Count failed authentication attempts
cat /var/log/mongodb/auditLog.json | \
  python3 -c "
import sys, json
lines = [json.loads(l) for l in sys.stdin]
failures = [l for l in lines if l.get('atype') == 'authenticate' and l.get('result') != 0]
print(f'Failed auth attempts: {len(failures)}')
for f in failures[:5]:
    print(f['remote']['ip'], f['param']['user'])
"
```

## Auditing to Syslog

For centralized log management, send audit events to syslog:

```yaml
auditLog:
  destination: syslog
  format: JSON
```

Then forward via rsyslog or syslog-ng to your SIEM.

## Summary

MongoDB Enterprise auditing is enabled by adding the `auditLog` section to `mongod.conf` with destination, format, and path. Use the `filter` option to reduce log volume by targeting specific operation types or namespaces. JSON format integrates well with log aggregation tools and SIEMs for compliance reporting and security incident investigation.
