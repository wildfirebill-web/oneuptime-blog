# How to Limit Concurrent IPv4 Connections per Client in Squid

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Squid, Connection Limiting, IPv4, ACL, Rate Limiting, Security, Configuration

Description: Learn how to configure Squid to limit the number of concurrent proxy connections from individual IPv4 client addresses to prevent resource exhaustion.

---

Without connection limits, a single misbehaving or compromised IPv4 client can consume all of Squid's worker connections. The `maxconn` ACL combined with `client_db` limits how many simultaneous connections each client IP may maintain.

## How Squid Tracks Connections per Client

Squid maintains a client database (`client_db`) that records per-IP statistics including active connection counts. The `maxconn` ACL type checks this database against a threshold.

## Basic Per-IP Connection Limit

```squid
# /etc/squid/squid.conf

# Enable the client database (required for maxconn ACLs)

client_db on

# --- ACL: flag clients with more than 10 concurrent connections ---
acl too_many_connections maxconn 10

# --- Deny requests from clients exceeding the limit ---
# Place this rule BEFORE the general allow rules
http_access deny too_many_connections

# --- General access control ---
acl localnet src 192.168.0.0/16
http_access allow localnet
http_access deny all
```

## Different Limits for Different Client Groups

```squid
client_db on

# Groups by source IPv4
acl power_users src 10.1.0.0/24     # Power users: higher limit
acl regular_users src 10.2.0.0/24   # Regular users: normal limit

# Connection limit ACLs
acl power_user_limit  maxconn 50    # Up to 50 concurrent connections
acl regular_user_limit maxconn 15   # Up to 15 concurrent connections

# Apply limits per group
http_access deny power_users power_user_limit
http_access deny regular_users regular_user_limit

# Allow after limits check passes
http_access allow power_users
http_access allow regular_users
http_access deny all
```

## Setting a System-Wide Maximum

```squid
# Limit total active connections to Squid from all clients
# This is a hard cap on Squid's file descriptor usage
# Default is 0 (unlimited)
max_filedesc 4096

# Recommended: set the connection limit via the OS too
# ulimit -n 4096  (or in systemd service file: LimitNOFILE=4096)
```

## Viewing Active Connection Counts

```bash
# View per-client connection statistics via the cache manager
squidclient -h localhost -p 3128 mgr:client_list

# Example output line:
# 192.168.1.5      Requests: 42   Connections: 8

# Watch active connections in real time
watch -n2 "squidclient -h localhost -p 3128 mgr:client_list | head -30"
```

## Logging When the Limit Is Hit

```bash
# Check access log for TCP_DENIED entries
grep "TCP_DENIED" /var/log/squid/access.log | grep "maxconn"

# Count denials per client IP
grep "TCP_DENIED" /var/log/squid/access.log | awk '{print $3}' | sort | uniq -c | sort -rn
```

## Key Takeaways

- `client_db on` is required for `maxconn` ACLs to work.
- Place `http_access deny too_many_connections` before general allow rules.
- Apply different `maxconn` limits to different IPv4 subnets for per-group policies.
- Use `squidclient mgr:client_list` to view current per-client connection counts in real time.
