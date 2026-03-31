# How to Configure MongoDB Log Verbosity Levels

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Logging, Configuration

Description: Learn how to configure MongoDB log verbosity levels at runtime and in mongod.conf to control the detail of diagnostic output for debugging.

---

MongoDB produces diagnostic logs that are invaluable for troubleshooting. Log verbosity controls how much detail appears in those logs. Setting verbosity too high floods your logs; too low leaves you without the information needed to diagnose problems.

## Default Verbosity

MongoDB uses verbosity level `0` by default, which logs informational messages and warnings. Levels range from `0` (least verbose) to `5` (most verbose).

## Setting Verbosity in mongod.conf

Edit `/etc/mongod.conf` to set global verbosity:

```text
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  verbosity: 1
```

You can also set component-specific verbosity:

```text
systemLog:
  component:
    query:
      verbosity: 2
    replication:
      verbosity: 1
    storage:
      verbosity: 0
```

Restart to apply file-based changes:

```bash
sudo systemctl restart mongod
```

## Setting Verbosity at Runtime

You can change verbosity without restarting by using `mongosh`:

```javascript
// Set global verbosity to level 1
db.setLogLevel(1)

// Set verbosity for a specific component
db.setLogLevel(2, "query")
db.setLogLevel(1, "replication")
db.setLogLevel(0, "storage")
```

## Checking Current Verbosity

To see the current log level settings:

```javascript
db.getLogComponents()
```

Sample output:

```text
{
  verbosity: 0,
  accessControl: { verbosity: -1 },
  command: { verbosity: -1 },
  query: { verbosity: 2 },
  replication: { verbosity: 1 },
  storage: { verbosity: 0 },
  ...
}
```

A value of `-1` means the component inherits the global verbosity level.

## Log Verbosity Components

Key components you can tune individually:

| Component | What it covers |
|-----------|---------------|
| `query` | Query planning and execution |
| `replication` | Replica set operations |
| `storage` | Storage engine activity |
| `network` | Network and connection details |
| `index` | Index build operations |
| `command` | Database command execution |

## Viewing Recent Log Messages

To retrieve the most recent log entries from `mongosh`:

```javascript
db.adminCommand({ getLog: "global" })
```

Or tail the log file directly:

```bash
tail -f /var/log/mongodb/mongod.log
```

## Best Practices

- Use level `0` in production to minimize log volume.
- Temporarily raise verbosity to `1` or `2` when troubleshooting specific components.
- Always reset verbosity after debugging to avoid filling your disk.

```javascript
// Reset all components to default
db.setLogLevel(0)
db.setLogLevel(-1, "query")
```

## Summary

MongoDB log verbosity can be configured globally or per-component either in `mongod.conf` or at runtime via `mongosh`. Use component-level settings to narrow diagnostic output during troubleshooting, and always revert to level 0 in production to keep log files manageable.
