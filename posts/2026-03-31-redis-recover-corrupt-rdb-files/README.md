# How to Recover Redis Data from Corrupt RDB Files

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RDB, Recovery, Data

Description: Use redis-check-rdb and manual repair techniques to recover data from corrupt Redis RDB snapshot files after disk failures or incomplete writes.

---

Redis RDB files can become corrupt due to incomplete writes, disk failures, or file system errors. The `redis-check-rdb` utility and a few manual techniques can often recover most or all of your data.

## Detecting RDB Corruption

Redis validates the RDB file on startup using a CRC64 checksum:

```text
# Redis startup log
Short read or OOM loading DB. Unrecoverable error, aborting now.
```

Or:

```text
DB loaded from disk: 0.000 seconds
```

If Redis refuses to start and the log mentions the RDB file, corruption is likely.

Check manually:

```bash
redis-check-rdb /var/lib/redis/dump.rdb
```

```text
[offset 0] Checking RDB file dump.rdb
[offset 26] AUX FIELD redis-ver = '7.0.5'
[offset 40] AUX FIELD redis-bits = '64'
[offset 52] AUX FIELD ctime = '1711900800'
[offset 64] AUX FIELD used-mem = '2147483648'
[offset 91] Selecting DB ID 0
[offset 100] Scanning type: RDB_TYPE_STRING (key: session:user1)
[err] Segment fault
```

An error mid-scan indicates the corruption location.

## Step 1: Never Modify the Original File

Always work on a copy:

```bash
cp /var/lib/redis/dump.rdb /var/lib/redis/dump.rdb.backup
cp /var/lib/redis/dump.rdb /tmp/dump-repair.rdb
```

## Step 2: Attempt Automatic Fix with redis-check-rdb

Redis includes a fix mode:

```bash
redis-check-rdb --fix /tmp/dump-repair.rdb
```

This truncates the file at the first corruption point, preserving all data before the corrupt section. Keys after the corruption are lost.

```text
Checking RDB file dump-repair.rdb
[offset 100] Scanning type: RDB_TYPE_STRING
[err] Corrupted data at offset 512
Fixing...
Truncated at 512 bytes.
```

## Step 3: Load the Repaired File

Test loading the repaired file:

```bash
# Start a temporary Redis instance with the repaired file
redis-server \
  --port 6399 \
  --dir /tmp \
  --dbfilename dump-repair.rdb \
  --daemonize yes \
  --logfile /tmp/redis-repair.log

# Check how many keys loaded
redis-cli -p 6399 DBSIZE

# Inspect keys
redis-cli -p 6399 KEYS "*" | head -20
```

## Step 4: Export Recovered Data

If the recovered dataset looks valid, export it:

```bash
# Dump all keys to a file for import into production
redis-cli -p 6399 --scan --pattern "*" | while read key; do
    type=$(redis-cli -p 6399 TYPE "$key")
    ttl=$(redis-cli -p 6399 TTL "$key")
    echo "TYPE: $type KEY: $key TTL: $ttl"
done
```

For a full migration, use `redis-cli --rdb`:

```bash
# Dump to a fresh RDB from the repair instance
redis-cli -p 6399 BGSAVE
cp /tmp/dump-repair.rdb /var/lib/redis/dump.rdb
```

## Step 5: Recover from a Replica or Backup

If the primary RDB is corrupt, the best recovery source is a replica or a recent backup:

```bash
# Copy a replica's RDB file to the primary
# First, trigger a save on the replica
redis-cli -h replica-host BGSAVE
sleep 10

# Copy the file
scp replica-host:/var/lib/redis/dump.rdb /var/lib/redis/dump.rdb

# Start primary Redis
sudo systemctl start redis
```

## Preventing Future Corruption

```text
# redis.conf
rdbchecksum yes       # Validate checksum on load
rdbcompression yes    # Smaller files, faster checksums
```

```bash
# Validate RDB file after every backup
redis-check-rdb /backup/redis/dump.rdb && echo "Backup is valid"
```

Use filesystem-level protection:

```bash
# Enable journaling on the Redis data filesystem
tune2fs -j /dev/sda1   # ext4 journaling

# Or use ZFS for copy-on-write protection
```

## Summary

Recover corrupt RDB files using `redis-check-rdb --fix` to truncate at the first corruption point and salvage all preceding data. Always work on a copy, never the original file. Load the repaired file in a temporary Redis instance to validate and extract data before replacing the production file. Enable `rdbchecksum yes` and validate backups with `redis-check-rdb` to detect corruption early.
