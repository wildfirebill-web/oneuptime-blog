# How to Configure Redis protected-mode for IPv4 Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, IPv4, Protected-mode, Security, Configuration, Cache

Description: Understand Redis protected-mode behavior, when to enable or disable it, and how to safely configure Redis for remote IPv4 access while maintaining security.

## Introduction

Redis `protected-mode` (introduced in 3.2) is a safety net that prevents remote access when Redis has no password and is bound to a non-loopback interface. When enabled, Redis replies to remote connections with an error message rather than allowing unauthenticated access. Understanding when to enable or disable it is essential for secure deployments.

## How protected-mode Works

```bash
# protected-mode = yes (default):

# If Redis:
#   - Is bound to a non-loopback address (i.e., 0.0.0.0 or 10.0.0.5)
#   AND
#   - Has no password (requirepass)
# Then: Remote connections get: "DENIED Redis is running in protected mode"

# protected-mode = no:
# Redis accepts remote connections regardless of password setting
# (Dangerous without requirepass and firewall!)
```

## When to Disable protected-mode

```bash
# SAFE to disable when:
# 1. requirepass is set AND
# 2. Firewall limits which IPs can reach port 6379

# /etc/redis/redis.conf

# Bind to specific IP
bind 127.0.0.1 10.0.0.5

# Set a strong password
requirepass "StrongPassword123!"

# Now safe to disable protected mode
protected-mode no
```

## When to Keep protected-mode Enabled

```bash
# KEEP enabled when:
# - Redis has no password
# - Used as a development environment
# - Deployment might accidentally expose Redis publicly

# /etc/redis/redis.conf
protected-mode yes     # Default - keep this for safety

# If you want remote access WITH protected-mode enabled,
# you must set requirepass:
requirepass "password"
# Redis will then allow authenticated remote connections even with protected-mode yes
```

## Protected Mode Behavior Matrix

| bind | requirepass | protected-mode | Remote access |
|---|---|---|---|
| 127.0.0.1 only | any | any | Local only |
| 0.0.0.0 | not set | yes | BLOCKED |
| 0.0.0.0 | set | yes | Allowed (authenticated) |
| 0.0.0.0 | not set | no | OPEN (dangerous!) |
| 10.0.0.5 | set | no | Allowed (authenticated) |

## Checking Current protected-mode Status

```bash
# Check configured value
redis-cli config get protected-mode

# Check if Redis is currently in protected mode
redis-cli -h 10.0.0.5 ping
# If in protected mode: DENIED Redis is running in protected mode...
# If auth required: NOAUTH Authentication required.
# If OK: PONG

# Change at runtime (doesn't persist to redis.conf)
redis-cli config set protected-mode no

# Check current settings
redis-cli info server | grep -E "protected|bind|requirepass"
```

## Recommended Production Configuration

```bash
# /etc/redis/redis.conf - production settings

# Bind to specific IPv4 only
bind 127.0.0.1 10.0.0.5

# Strong authentication password
requirepass "Use-A-Long-Random-Password-Here!"

# Can disable protected-mode since both above are set
protected-mode no

# Additional security
rename-command FLUSHALL ""    # Disable dangerous commands
rename-command CONFIG  ""
rename-command DEBUG   ""

# TCP listen backlog
tcp-backlog 511
```

```bash
# Restrict at firewall level (defense in depth)
sudo iptables -A INPUT -p tcp --dport 6379 -s 10.0.0.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6379 -j DROP
```

## Conclusion

Redis `protected-mode` is a safety guard that prevents unauthenticated remote access. Disable it only when `requirepass` is set AND firewall rules restrict port 6379 to trusted networks. Leave it enabled in development environments or when you're unsure about network exposure. The protected-mode error message is a warning that Redis is accessible from an unexpected location without authentication.
