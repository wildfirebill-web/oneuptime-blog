# How to Configure Redis rename-command for Security

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Security, Configuration

Description: Use Redis rename-command to disable or obscure dangerous commands like FLUSHALL, DEBUG, and CONFIG to reduce the attack surface on production servers.

---

Redis includes powerful administrative commands that can be destructive in the wrong hands. The `rename-command` directive in `redis.conf` lets you rename or completely disable commands to limit exposure.

## How rename-command Works

The directive takes two arguments: the original command name and the new name. To disable a command entirely, rename it to an empty string:

```text
# redis.conf

# Disable FLUSHALL completely
rename-command FLUSHALL ""

# Rename DEBUG to an obscure string
rename-command DEBUG "a8f3c2b91e4d7f6a"

# Disable CONFIG on untrusted ports
rename-command CONFIG ""
```

After restarting Redis, calling `FLUSHALL` returns an error:

```text
(error) ERR unknown command `FLUSHALL`
```

## Commonly Disabled Commands

These commands are most often restricted in production:

| Command | Risk |
| --- | --- |
| FLUSHALL | Deletes all data in all databases |
| FLUSHDB | Deletes all data in the current database |
| DEBUG | Can cause crashes, trigger slowlog, change object encoding |
| CONFIG | Can overwrite config files or change memory limits |
| KEYS | Blocks server during full keyspace scan |
| SHUTDOWN | Stops the Redis process |
| REPLICAOF | Can change replication topology |

Example configuration that hardens a public-facing Redis:

```text
# redis.conf - hardened configuration
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command DEBUG ""
rename-command CONFIG "CONFIG_INTERNAL_9f3a2b"
rename-command SHUTDOWN "SHUTDOWN_INTERNAL_7e1c4d"
rename-command KEYS "KEYS_INTERNAL_2d8f5e"
rename-command REPLICAOF ""
```

## Applying the Rename at Runtime

You cannot use `CONFIG SET` to change `rename-command` - it must be set in `redis.conf` before startup. Changes require a Redis restart:

```bash
sudo nano /etc/redis/redis.conf
# Add rename-command directives

sudo systemctl restart redis
```

## Verifying the Configuration

After restarting, verify the commands are disabled:

```bash
redis-cli FLUSHALL
# Expected: (error) ERR unknown command `FLUSHALL`

redis-cli CONFIG GET maxmemory
# Expected: (error) ERR unknown command `CONFIG` (if disabled)
```

If you renamed CONFIG, use the new name in your application and admin scripts:

```bash
redis-cli CONFIG_INTERNAL_9f3a2b GET maxmemory
```

## Using ACLs as a Modern Alternative

Since Redis 6.0, ACL rules provide more granular and dynamic control than `rename-command`. You can restrict commands per user without restarting:

```bash
# Restrict a user from running FLUSHALL
redis-cli ACL SETUSER appuser on >password ~app:* +@all -FLUSHALL -FLUSHDB -DEBUG

# Check what commands appuser can run
redis-cli ACL GETUSER appuser
```

ACLs are preferable for multi-user setups. Use `rename-command` as an additional layer when you want to prevent accidental or malicious use even by privileged users.

## Impact on Replicas and Cluster

If you rename commands on a primary, replicas receive the renamed command over the replication stream. Ensure all nodes in a replica set or cluster have consistent `rename-command` settings to avoid replication errors.

```text
# Both primary and replica redis.conf must match
rename-command FLUSHALL ""
rename-command DEBUG ""
```

## Summary

`rename-command` in `redis.conf` is a simple and effective way to disable or obscure dangerous commands like FLUSHALL, DEBUG, and CONFIG on production Redis servers. For dynamic, per-user access control, combine it with ACL rules introduced in Redis 6. Always ensure consistent rename-command settings across all nodes in a replication setup.
