# How to Configure Audit Options in mongod.conf

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Audit, Security, Configuration, mongod.conf, Compliance

Description: Learn how to configure MongoDB's built-in audit logging in mongod.conf to record authentication, authorization, and data access events for compliance.

---

## Introduction

MongoDB Enterprise includes a built-in audit log facility that records database operations for security and compliance purposes. Audit logging can capture authentication events, CRUD operations, schema changes, and privilege escalations. The `auditLog` section in `mongod.conf` controls what is captured and where it is stored.

## Prerequisites

Audit logging requires MongoDB Enterprise. Verify with:

```bash
mongod --version | grep "enterprise"
```

## Basic Audit Configuration

Write audit events to a JSON file:

```yaml
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/auditLog.json
```

## Audit Destinations

| Destination | Description |
|------------|-------------|
| file | Write to a file (JSON or BSON) |
| console | Write to stdout |
| syslog | Write to system syslog |

## Filtering Audit Events

Use a filter to capture only specific actions:

```yaml
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/auditLog.json
  filter: '{ atype: { $in: ["authenticate", "authCheck", "logout", "createCollection", "dropCollection", "createIndex", "dropDatabase"] } }'
```

## Capturing Authentication Events Only

For a minimal audit footprint focused on security events:

```yaml
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/auditLog.json
  filter: '{ atype: { $in: ["authenticate", "logout", "createUser", "dropUser", "updateUser"] } }'
```

## Capturing Data Access for Compliance

For HIPAA or PCI-DSS compliance, capture read and write operations on sensitive collections:

```yaml
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/auditLog.json
  filter: '{ atype: "authCheck", "param.ns": { $in: ["mydb.patients", "mydb.payments"] } }'
```

## Sample Audit Log Entry

A typical JSON audit log entry looks like:

```json
{
  "atype": "authenticate",
  "ts": { "$date": "2026-03-31T12:00:00.000Z" },
  "local": { "ip": "127.0.0.1", "port": 27017 },
  "remote": { "ip": "192.168.1.10", "port": 54321 },
  "users": [{ "user": "alice", "db": "admin" }],
  "roles": [{ "role": "readWrite", "db": "mydb" }],
  "param": { "user": "alice", "db": "admin", "mechanism": "SCRAM-SHA-256" },
  "result": 0
}
```

## Rotating Audit Logs

Audit log files grow continuously. Rotate them using logrotate:

```bash
# /etc/logrotate.d/mongodb-audit
/var/log/mongodb/auditLog.json {
  daily
  rotate 90
  compress
  missingok
  notifempty
  postrotate
    kill -SIGUSR1 $(cat /var/run/mongodb/mongod.pid)
  endscript
}
```

## Summary

MongoDB Enterprise's built-in audit logging provides granular visibility into database activity for security and compliance. Configuring the `auditLog` section in `mongod.conf` with appropriate filters limits audit volume while capturing the events that matter. Combining audit logs with log rotation and long-term storage satisfies most regulatory retention requirements.
