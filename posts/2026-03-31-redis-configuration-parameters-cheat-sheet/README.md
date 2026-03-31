# Redis Configuration Parameters Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Configuration, Cheat Sheet, Performance, Security

Description: Complete Redis configuration parameters reference covering memory, persistence, networking, security, replication, and performance tuning.

---

Redis is configured via `redis.conf` or at runtime using `CONFIG SET`. Here is a complete reference of the most important configuration parameters organized by category.

## Networking

```text
bind 127.0.0.1 -::1         # bind to interfaces (use specific IPs in prod)
port 6379                    # TCP port
unixsocket /tmp/redis.sock   # Unix socket path
unixsocketperm 700           # socket file permissions
timeout 0                    # close idle client connections (0 = never)
tcp-keepalive 300            # send TCP keepalives every N seconds
tcp-backlog 511              # length of SYN backlog queue
maxclients 10000             # max simultaneous connections
```

## Memory

```text
maxmemory 2gb                # max memory usage
maxmemory-policy allkeys-lru # eviction policy when maxmemory reached
maxmemory-samples 5          # samples for LRU/LFU approximation

# Eviction policies:
# noeviction         - return errors (default)
# allkeys-lru        - evict least recently used from all keys
# volatile-lru       - LRU only among keys with TTL
# allkeys-lfu        - evict least frequently used
# volatile-lfu       - LFU only among keys with TTL
# allkeys-random     - random eviction
# volatile-random    - random among keys with TTL
# volatile-ttl       - evict key with nearest TTL

active-expire-enabled yes    # proactive key expiration
lazyfree-lazy-eviction no    # async eviction (yes = non-blocking)
lazyfree-lazy-expire no      # async key expiration
lazyfree-lazy-server-del no  # async DEL operations
```

## Persistence - RDB

```text
save 3600 1     # save if 1 key changed in 3600 seconds
save 300 100    # save if 100 keys changed in 300 seconds
save 60 10000   # save if 10000 keys changed in 60 seconds
save ""         # disable RDB snapshots

dbfilename dump.rdb          # RDB file name
dir /var/lib/redis           # working directory
rdbcompression yes           # compress RDB with LZF
rdbchecksum yes              # CRC64 checksum for RDB
```

## Persistence - AOF

```text
appendonly no                # enable AOF (yes/no)
appendfilename appendonly.aof
appendfsync everysec         # fsync policy: always, everysec, no
no-appendfsync-on-rewrite no # skip fsync during AOF rewrite
auto-aof-rewrite-percentage 100  # trigger rewrite at 100% growth
auto-aof-rewrite-min-size 64mb   # minimum size before rewrite
aof-use-rdb-preamble yes     # faster AOF restart with RDB prefix
```

## Replication

```text
replicaof 192.168.1.100 6379 # make this node a replica
masterauth password           # password to authenticate to primary
replica-read-only yes         # replicas are read-only
replica-lazy-flush no         # async replica flush on SYNC
repl-backlog-size 1mb         # replication buffer size
repl-backlog-ttl 3600         # how long to keep backlog after no replicas
min-replicas-to-write 1       # min replicas needed for writes
min-replicas-max-lag 10       # max replication lag in seconds
```

## Security

```text
requirepass your_strong_password
aclfile /etc/redis/users.acl    # ACL file path
acllog-max-len 128              # max ACL log entries
rename-command FLUSHALL ""      # disable dangerous commands
rename-command CONFIG ""
protected-mode yes              # block external connections if no bind/auth
```

## TLS (Redis 6.0+)

```text
tls-port 6380
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-auth-clients yes     # require client certificates
tls-replication yes      # use TLS for replication
tls-cluster yes          # use TLS for cluster bus
```

## Performance

```text
hz 10                    # server background task frequency (10-500)
dynamic-hz yes           # increase hz under load
activerehashing yes      # rehash main dictionary during idle time
lazyfree-lazy-user-del no # async user-triggered DEL
io-threads 4             # I/O threads (Redis 6.0+, usually 4-8)
io-threads-do-reads yes  # also use I/O threads for reads
```

## Slow Log

```text
slowlog-log-slower-than 10000  # log commands slower than N microseconds
slowlog-max-len 128            # max entries in slow log
```

## Cluster

```text
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 15000      # node failure timeout in ms
cluster-require-full-coverage yes  # refuse writes if slots are uncovered
cluster-migration-barrier 1    # min replicas for slot migration
```

## Summary

Redis configuration parameters control memory limits and eviction, persistence via RDB snapshots and AOF, replication settings, security (passwords, ACLs, TLS), and performance tuning. Most parameters can be changed at runtime with `CONFIG SET` without restarting, and saved back to `redis.conf` with `CONFIG REWRITE`.
