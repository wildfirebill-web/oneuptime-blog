# How to Configure pg_hba.conf for IPv4 Host-Based Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: PostgreSQL, pg_hba.conf, IPv4, Authentication, Access Control, Database

Description: Configure PostgreSQL's pg_hba.conf to control which IPv4 hosts can connect, which users and databases they can access, and which authentication method to use.

## Introduction

`pg_hba.conf` is PostgreSQL's host-based authentication file. Every incoming connection is matched against its rules from top to bottom; the first matching rule determines whether access is granted. Rules specify connection type, database, user, client address, and authentication method.

## pg_hba.conf Rule Format

```bash
# Format:

# TYPE  DATABASE  USER  ADDRESS  METHOD

# TYPE values:
# local    - Unix domain socket
# host     - TCP/IP (SSL or non-SSL)
# hostssl  - TCP/IP with SSL required
# hostnossl - TCP/IP without SSL

# METHOD values:
# trust      - Accept without password (local only!)
# reject     - Reject always
# md5        - MD5 password
# scram-sha-256 - SCRAM-SHA-256 (recommended for PostgreSQL 14+)
# peer       - OS username must match (local only)
# ident      - Ident server (rare)
```

## Common pg_hba.conf Patterns

```bash
# /etc/postgresql/16/main/pg_hba.conf

# Local connections (Unix socket) - trust for postgres user
local   all             postgres                                peer

# Local IPv4 loopback
host    all             all             127.0.0.1/32            scram-sha-256

# Allow specific user to specific database from specific IP
host    appdb           appuser         10.0.0.5/32             scram-sha-256

# Allow all users to all DBs from a subnet
host    all             all             10.0.0.0/24             md5

# Allow SSL-only from DMZ
hostssl all             all             203.0.113.0/24          scram-sha-256

# Allow read-only replica user from replication server
host    replication     replicator      10.0.0.10/32            scram-sha-256

# Block specific IP explicitly (before subnet allow rule)
host    all             all             10.0.0.99/32            reject

# Deny everything else
host    all             all             0.0.0.0/0               reject
```

## Rule Ordering Matters

```bash
# WRONG: Subnet ALLOW before specific IP DENY - specific IP still allowed!
host all all 10.0.0.0/24 md5
host all all 10.0.0.99/32 reject   # Never reached for 10.0.0.99

# CORRECT: Specific DENY before subnet ALLOW
host all all 10.0.0.99/32 reject
host all all 10.0.0.0/24 md5       # 10.0.0.99 already rejected
```

## Applying Changes

```bash
# After editing pg_hba.conf, reload PostgreSQL
sudo systemctl reload postgresql

# Or from psql
sudo -u postgres psql -c "SELECT pg_reload_conf();"

# Verify the reload happened
sudo -u postgres psql -c "SELECT now();" postgres   # Quick connection test
```

## Testing Authentication Rules

```bash
# Test connection as a specific user from localhost
psql -h 127.0.0.1 -U appuser -d appdb

# Test from a specific IP (simulate from another host)
psql -h 10.0.0.5 -U appuser -d appdb

# View pg_hba.conf in PostgreSQL
sudo -u postgres psql -c "SELECT * FROM pg_hba_file_rules;"

# View connection log for auth failures
sudo tail -f /var/log/postgresql/postgresql-16-main.log | grep -E "FATAL|auth"
```

## Conclusion

`pg_hba.conf` rules are matched from top to bottom-order is critical. Place `reject` rules before `allow` rules when blocking specific IPs within an allowed subnet. Use `scram-sha-256` over `md5` for new PostgreSQL 14+ deployments. Always use `hostssl` for connections from untrusted networks, and reload (not restart) PostgreSQL after pg_hba.conf changes.
