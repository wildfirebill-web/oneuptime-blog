# How to Troubleshoot Redis Fork Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Fork, RDB, Persistence, Troubleshooting

Description: Fix Redis fork failures caused by insufficient memory, overcommit settings, or too many processes - including BGSAVE and BGREWRITEAOF fork errors.

---

Redis uses `fork()` to create a child process for RDB snapshots (`BGSAVE`) and AOF rewrites (`BGREWRITEAOF`). If `fork()` fails, you see errors like `Can't save in background: fork: Cannot allocate memory` or `MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk`. This guide explains the causes and fixes.

## Understanding Why Fork Fails

When Redis forks, the OS needs to have enough virtual memory available. Even though copy-on-write (COW) means the child does not immediately copy all memory, the kernel must be able to commit the virtual address space.

Two main causes:
1. **Overcommit is disabled** - kernel refuses `fork()` because it cannot guarantee the memory theoretically needed
2. **Actual memory exhaustion** - system has no free RAM or swap

```bash
# Check current Redis memory usage
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human"

# Check system available memory
free -h

# Check if a fork is currently running
redis-cli INFO persistence | grep -E "rdb_bgsave_in_progress|aof_rewrite_in_progress"
```

## Diagnose the Error

```bash
# Check Redis logs for fork errors
sudo journalctl -u redis -n 100 | grep -i fork
sudo grep -i "fork\|bgsave\|Cannot allocate" /var/log/redis/redis-server.log

# Check Redis persistence status
redis-cli INFO persistence | grep -E "rdb_last_bgsave_status|aof_last_bgrewrite_status"
# If showing: rdb_last_bgsave_status:err

# Manually trigger BGSAVE and check for errors
redis-cli BGSAVE
redis-cli LASTSAVE
redis-cli INFO persistence | grep rdb_last_bgsave_status
```

## Fix 1: Enable Memory Overcommit

This is the most common fix. Linux's strict memory accounting rejects `fork()` even when there is plenty of RAM with copy-on-write:

```bash
# Check current setting
cat /proc/sys/vm/overcommit_memory
# 0 = heuristic (default, can reject fork)
# 1 = always allow (recommended for Redis)
# 2 = strict (never allow overcommit)

# Enable overcommit temporarily
sudo sysctl -w vm.overcommit_memory=1

# Make permanent
echo "vm.overcommit_memory = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Verify
cat /proc/sys/vm/overcommit_memory
# Should show: 1
```

## Fix 2: Check and Increase Swap Space

```bash
# Check swap usage
free -h
swapon --show

# If swap is nearly full, add more swap space
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent
echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab

# Verify
free -h
```

## Fix 3: Reduce maxmemory or Data Size

If the system genuinely cannot support the fork (e.g., a container with strict memory limits):

```bash
# Reduce Redis memory footprint
redis-cli CONFIG SET maxmemory 2gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# Check how much memory is actually being used
redis-cli INFO memory | grep used_memory_human

# Remove large keys
redis-cli --bigkeys
redis-cli DEL large-key-name
```

## Fix 4: Disable RDB Saves if Persistence Is Not Needed

If Redis is used purely as a cache and persistence is not required:

```bash
# Disable all RDB saves
redis-cli CONFIG SET save ""

# Disable AOF
redis-cli CONFIG SET appendonly no

# Persist to redis.conf
redis-cli CONFIG REWRITE
```

## Fix 5: Check Fork Limitations

```bash
# Check max processes limit
cat /proc/sys/kernel/pid_max
ulimit -u

# Check if too many processes are running
ps aux | wc -l

# Increase process limits if needed
echo "kernel.pid_max = 4194304" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Monitor Fork Operations

```bash
# Watch for fork time - long fork times indicate memory pressure
redis-cli INFO stats | grep latest_fork_usec
# Example: latest_fork_usec:1234  <- 1.2ms fork time
# > 20ms is concerning; > 100ms is a serious problem

# Monitor persistence status continuously
watch -n5 'redis-cli INFO persistence | grep -E "bgsave|aof_rewrite"'
```

## Fix: Transparent Huge Pages Causing Fork Slowness

Even if fork does not fail, THP can make it very slow:

```bash
# Check THP status
cat /sys/kernel/mm/transparent_hugepage/enabled

# Disable THP
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

# Add to /etc/rc.local for persistence across reboots
```

## Verify Fork is Working

```bash
# Trigger BGSAVE and verify it succeeds
redis-cli BGSAVE
# Expected: Background saving started

# Wait and check status
sleep 3
redis-cli INFO persistence | grep rdb_last_bgsave_status
# Expected: rdb_last_bgsave_status:ok

# Check fork time
redis-cli INFO stats | grep latest_fork_usec
```

## Summary

Redis fork failures are almost always caused by `vm.overcommit_memory=0` on Linux. Set it to 1 to allow fork to succeed even when the kernel cannot guarantee the theoretical memory requirement. For containers with strict memory limits, reduce Redis's `maxmemory` setting or disable RDB persistence if the data is purely cache. Monitor `latest_fork_usec` to catch slow fork operations before they become failures.
