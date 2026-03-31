# How to Recover Redis Data from Corrupt AOF Files

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, AOF, Recovery, Data

Description: Use redis-check-aof to detect and repair corrupt AOF files, truncating incomplete commands to restore Redis data after crashes or disk errors.

---

The Redis Append-Only File (AOF) can become corrupt when a crash occurs mid-write, leaving a partial command at the end of the file. The `redis-check-aof` utility detects and repairs these issues to restore your data.

## What AOF Corruption Looks Like

When Redis starts and finds a corrupt AOF file:

```text
# Redis startup log
Bad file format reading the append only file: make a backup of your AOF file, then use ./redis-check-aof --fix <filename>
```

Redis refuses to load a corrupt AOF to prevent serving inconsistent data.

## Step 1: Back Up the Corrupt File

Never modify the original:

```bash
cp /var/lib/redis/appendonly.aof /var/lib/redis/appendonly.aof.backup
cp /var/lib/redis/appendonly.aof /tmp/appendonly-repair.aof
```

## Step 2: Diagnose the Corruption

Run `redis-check-aof` without `--fix` first to see what is wrong:

```bash
redis-check-aof /tmp/appendonly-repair.aof
```

```text
Checking Redis AOF file /tmp/appendonly-repair.aof
AOF analyzed: size=52428800, ok_up_to=52427800, diff=1000
This AOF file is not valid. The last 1000 bytes are unreachable.
```

The output shows:
- `ok_up_to` - how many bytes are valid
- `diff` - how many bytes are corrupt or incomplete

## Step 3: Fix the AOF File

```bash
redis-check-aof --fix /tmp/appendonly-repair.aof
```

```text
Checking Redis AOF file /tmp/appendonly-repair.aof
AOF analyzed: size=52428800, ok_up_to=52427800, diff=1000
This will trim the AOF from 52428800 bytes to 52427800 bytes
Continue? [y/N]: y
Successfully truncated AOF
```

The tool truncates the file at the last complete command, discarding the incomplete tail. Data up to that point is preserved.

## Step 4: Verify the Repaired File

```bash
# Check the repaired file passes validation
redis-check-aof /tmp/appendonly-repair.aof
```

```text
Checking Redis AOF file /tmp/appendonly-repair.aof
AOF analyzed: size=52427800, ok_up_to=52427800, diff=0
AOF is valid
```

`diff=0` means the entire file is valid.

## Step 5: Test Loading the Repaired File

Start a temporary Redis instance with the repaired file:

```bash
redis-server \
  --port 6399 \
  --dir /tmp \
  --appendonly yes \
  --appendfilename appendonly-repair.aof \
  --daemonize yes \
  --logfile /tmp/redis-repair.log

# Check key count
redis-cli -p 6399 DBSIZE

# Verify some critical keys
redis-cli -p 6399 GET "important:key"
```

## Step 6: Replace Production AOF

Once verified:

```bash
# Stop production Redis
sudo systemctl stop redis

# Replace the corrupt AOF with the repaired version
cp /tmp/appendonly-repair.aof /var/lib/redis/appendonly.aof
sudo chown redis:redis /var/lib/redis/appendonly.aof

# Start Redis
sudo systemctl start redis

# Verify startup
redis-cli PING
redis-cli DBSIZE
```

## Handling Hybrid AOF Files

If `aof-use-rdb-preamble` is enabled, the AOF starts with an RDB section. The `redis-check-aof` tool handles this automatically, but you can verify:

```bash
xxd /var/lib/redis/appendonly.aof | head -2
```

```text
00000000: 5245 4449 5330 3031 31...  REDIS0011
```

If the RDB preamble itself is corrupt, use `redis-check-rdb` logic to recover the snapshot portion first.

## Prevention: AOF Durability Settings

To prevent corruption from incomplete writes:

```text
# redis.conf
appendfsync everysec        # fsync every second
aof-use-rdb-preamble yes    # compact rewrites reduce AOF size
```

Validate AOF files in your backup process:

```bash
#!/bin/bash
AOF=/backup/redis/appendonly-$(date +%Y%m%d).aof
cp /var/lib/redis/appendonly.aof "$AOF"

if redis-check-aof "$AOF" | grep -q "AOF is valid"; then
    echo "Backup valid: $AOF"
else
    echo "ALERT: AOF backup is corrupt"
fi
```

## Summary

Recover corrupt AOF files with `redis-check-aof --fix`, which truncates the file at the last complete command. Back up the original before any repair, test the repaired file in a temporary Redis instance, and validate key counts before replacing the production file. Use `appendfsync everysec` and regular validated backups to minimize data loss and catch corruption early.
