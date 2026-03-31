# How to Rotate MongoDB Log Files

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mongodb, Logging, Operations, Database Administration

Description: Learn how to rotate MongoDB log files using the logRotate command, SIGUSR1 signal, and logRotate utility to manage log file size in production.

---

## Overview

As MongoDB runs over time, its log file grows continuously. Without log rotation, log files can consume excessive disk space and become difficult to navigate. MongoDB provides several built-in mechanisms for log rotation that allow you to manage log files without restarting the database.

## Prerequisites

Before rotating logs, ensure MongoDB is configured to log to a file:

```yaml
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: rename
```

The `logAppend: true` setting ensures MongoDB appends to the existing log file on restart. The `logRotate` setting controls behavior during rotation - either `rename` (default) or `reopen`.

## Method 1 - Using the logRotate Command in mongosh

You can trigger log rotation from within the MongoDB shell:

```javascript
// Connect to mongosh and run
use admin
db.adminCommand({ logRotate: 1 })
```

This renames the current log file by appending a timestamp and opens a new log file at the original path.

## Method 2 - Using the SIGUSR1 Signal on Linux

On Linux, send a SIGUSR1 signal to the mongod process to trigger log rotation:

```bash
# Find the mongod process ID
cat /var/run/mongodb/mongod.pid

# Send SIGUSR1 to rotate logs
kill -SIGUSR1 $(cat /var/run/mongodb/mongod.pid)

# Or use pkill
pkill -SIGUSR1 mongod
```

## Method 3 - Using the logRotate Utility

For automated log rotation, use the system `logrotate` utility. Create a configuration file:

```text
# /etc/logrotate.d/mongodb
/var/log/mongodb/mongod.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        /bin/kill -SIGUSR1 $(cat /var/run/mongodb/mongod.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
```

Test the logrotate configuration without actually rotating:

```bash
logrotate --debug /etc/logrotate.d/mongodb
```

Force rotation manually:

```bash
logrotate --force /etc/logrotate.d/mongodb
```

## The rename vs reopen Log Rotate Behaviors

MongoDB supports two log rotation behaviors controlled by `logRotate` in the config file:

**rename (default):** The current log file is renamed with a timestamp suffix, and a new file is created at the original path.

```bash
# Result after rename rotation:
# /var/log/mongodb/mongod.log          <- new file
# /var/log/mongodb/mongod.log.2026-03-31T10-00-00  <- old file
```

**reopen:** MongoDB closes and reopens the log file descriptor. This works with external tools like logrotate that rename the file first:

```yaml
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logRotate: reopen
```

Use `reopen` when integrating with the system `logrotate` utility.

## Verifying Log Rotation

After rotation, verify the new log file is being written to:

```bash
# Watch the log file in real time
tail -f /var/log/mongodb/mongod.log

# Check file timestamps
ls -lh /var/log/mongodb/

# Confirm mongod is writing to the new file
lsof -p $(cat /var/run/mongodb/mongod.pid) | grep log
```

## Automating Log Rotation with Cron

For systems without logrotate, set up a cron job:

```bash
# Add to crontab - rotate logs daily at midnight
0 0 * * * /bin/kill -SIGUSR1 $(cat /var/run/mongodb/mongod.pid) 2>/dev/null
```

## Summary

MongoDB log rotation can be performed using the `logRotate` admin command in mongosh, by sending a SIGUSR1 signal to the mongod process, or by integrating with the system logrotate utility. The `logRotate: reopen` configuration option is best for integration with external log management tools. Regular log rotation is essential for maintaining disk space and keeping logs manageable in production environments.
