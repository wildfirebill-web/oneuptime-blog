# How to Perform a Clean Shutdown of MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Administration, Operations

Description: Learn how to safely shut down a MongoDB instance using systemctl, mongosh, and the shutdown command to avoid data corruption and journal recovery.

---

An unclean MongoDB shutdown can trigger journal recovery on the next startup, slowing your restart time. In extreme cases it may cause data file corruption. This guide explains how to shut down MongoDB safely in various scenarios.

## Method 1: Using systemctl (Linux)

For deployments managed by systemd, this is the recommended approach:

```bash
sudo systemctl stop mongod
```

systemd sends a `SIGTERM` signal to `mongod`, which allows it to complete in-flight writes, flush dirty pages, and close cleanly.

Check that the service stopped:

```bash
sudo systemctl status mongod
```

## Method 2: Using the shutdown Command in mongosh

Connect to MongoDB with an admin user and issue the shutdown command:

```javascript
use admin
db.shutdownServer()
```

To force shutdown even if there are open transactions:

```javascript
db.shutdownServer({ force: true })
```

To add a timeout (wait up to 30 seconds for replica set members to catch up before shutting down):

```javascript
db.shutdownServer({ timeoutSecs: 30 })
```

## Method 3: Using kill with SIGTERM

If systemd is not available, send `SIGTERM` to the `mongod` process:

```bash
# Find the mongod PID
cat /var/run/mongodb/mongod.pid

# Send SIGTERM
kill -SIGTERM <pid>
```

Never use `SIGKILL` (`kill -9`) unless absolutely necessary, as it bypasses journal flushing.

## Shutting Down a Replica Set Primary

Before shutting down a primary, step it down to allow a secondary to be elected:

```javascript
use admin
rs.stepDown(60)  // Step down for 60 seconds
```

Once a new primary is elected, shut down the old primary:

```javascript
db.shutdownServer()
```

This avoids an election timeout period that would make the replica set read-only.

## Verifying a Clean Shutdown

After starting MongoDB again, check the log for the absence of recovery messages:

```bash
sudo journalctl -u mongod --since "5 minutes ago" | grep -i "recovery"
```

A clean startup shows:

```text
[initandlisten] wiredtiger_open config: ...
[initandlisten] WiredTiger message...
[initandlisten] Opened the WiredTiger journal to sequence number 123456
```

Without recovery messages, the previous shutdown was clean.

## Scheduled Maintenance Shutdowns

For planned maintenance, follow this sequence:

```bash
# 1. Drain connections (optional, in your app)
# 2. Step down if primary
# 3. Stop the service
sudo systemctl stop mongod

# 4. Perform maintenance
# 5. Restart
sudo systemctl start mongod
```

## Summary

Always shut down MongoDB using `systemctl stop mongod` or `db.shutdownServer()` to ensure a clean close. For replica set primaries, step down first to minimize disruption. Avoid `SIGKILL` as it bypasses journal flushing and risks data corruption.
