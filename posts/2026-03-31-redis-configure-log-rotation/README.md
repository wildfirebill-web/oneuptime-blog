# How to Configure Redis Log Rotation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Log Rotation, logrotate, Linux, Operations

Description: Configure log rotation for Redis using logrotate to prevent Redis log files from growing unbounded, with daily rotation, compression, and signal-based log reopening.

---

Redis logs to a file in production, and without log rotation that file can grow until it fills the disk. The standard tool for managing log rotation on Linux is `logrotate`, which is already installed on most distributions.

## Check the Current Log File Location

```bash
redis-cli CONFIG GET logfile
# 1) "logfile"
# 2) "/var/log/redis/redis-server.log"
```

If `logfile` is empty, Redis is logging to stdout. Set it in redis.conf:

```text
logfile /var/log/redis/redis-server.log
```

Then restart Redis.

## Create a logrotate Configuration

```bash
sudo tee /etc/logrotate.d/redis > /dev/null <<'EOF'
/var/log/redis/redis-server.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 640 redis redis
    sharedscripts
    postrotate
        redis-cli CONFIG SET loglevel notice
        redis-cli DEBUG SLEEP 0
    endscript
}
EOF
```

Options explained:
- `daily` - rotate once per day.
- `rotate 14` - keep 14 rotated log files.
- `compress` - gzip rotated files.
- `delaycompress` - compress the previous rotation, not the current one (allows the file to be fully closed first).
- `missingok` - no error if the log file is missing.
- `notifempty` - do not rotate empty files.
- `create 640 redis redis` - create the new log file with correct permissions.

## Sending a Signal to Redis to Reopen the Log File

Redis does not automatically reopen its log file after rotation. The standard approach is to use `CONFIG REWRITE` or send a `SIGUSR1` signal which causes Redis to reopen the log:

```bash
sudo tee /etc/logrotate.d/redis > /dev/null <<'EOF'
/var/log/redis/redis-server.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 640 redis redis
    sharedscripts
    postrotate
        /bin/kill -USR1 $(cat /var/run/redis/redis-server.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
EOF
```

If Redis does not write a PID file, use:

```bash
postrotate
    pkill -USR1 redis-server || true
endscript
```

## Enable a PID File in Redis

Add to redis.conf:

```text
pidfile /var/run/redis/redis-server.pid
```

Create the directory:

```bash
sudo mkdir -p /var/run/redis
sudo chown redis:redis /var/run/redis
```

## Test the logrotate Configuration

```bash
# Dry run - shows what would happen
sudo logrotate --debug /etc/logrotate.d/redis

# Force rotation (even if not due)
sudo logrotate --force /etc/logrotate.d/redis

# Verify rotation happened
ls -lh /var/log/redis/
```

## Rotating Multiple Redis Instances

If you run multiple Redis instances with separate log files, add them to the same logrotate config:

```text
/var/log/redis/redis-server.log
/var/log/redis/redis-replica.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        pkill -USR1 redis-server || true
    endscript
}
```

## Summary

Configure Redis log rotation with logrotate by creating `/etc/logrotate.d/redis` with daily rotation, 14-file retention, and gzip compression. Send a `SIGUSR1` signal in the `postrotate` script so Redis reopens the log file after rotation. Enable a PID file in `redis.conf` for reliable signal targeting, and use `logrotate --force` to test your configuration without waiting for the scheduled run.
