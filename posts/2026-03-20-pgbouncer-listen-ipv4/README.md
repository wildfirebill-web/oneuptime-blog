# How to Configure PgBouncer to Listen on a Specific IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: PgBouncer, PostgreSQL, IPv4, Connection Pooling, Database, Configuration

Description: Configure PgBouncer connection pooler to listen on a specific IPv4 address and port, set pooling mode, configure authentication, and tune pool sizes for production use.

## Introduction

PgBouncer is a lightweight connection pooler for PostgreSQL. By default it may listen on all interfaces. In production, you should bind it to the specific IPv4 address that application servers use to connect, keeping it off public interfaces.

## Basic pgbouncer.ini Configuration

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
# Database alias = connection string to PostgreSQL

myapp = host=127.0.0.1 port=5432 dbname=myapp_production

# Connect to a remote PostgreSQL server
analytics = host=10.0.0.20 port=5432 dbname=analytics user=pgbouncer

[pgbouncer]
# Listen on specific IPv4 address only
listen_addr = 10.0.0.5
listen_port = 6432

# Pooling mode: session, transaction, or statement
# transaction is most common for web apps
pool_mode = transaction

# Max connections to PostgreSQL (should be <= max_connections in postgresql.conf)
max_client_conn = 1000
default_pool_size = 25

# Authentication
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Logging
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid

# Admin interface (for pgbouncer SHOW STATS etc.)
# Listen on loopback only for admin
admin_users = pgbouncer_admin
stats_users = pgbouncer_monitor
```

## Multiple Listen Addresses

```ini
[pgbouncer]
# Listen on multiple IPv4 addresses (comma-separated)
listen_addr = 10.0.0.5, 10.0.1.5
listen_port = 6432

# Listen on all interfaces (not recommended for production)
# listen_addr = 0.0.0.0
```

## Authentication File

```bash
# /etc/pgbouncer/userlist.txt
# Format: "username" "scram-sha-256$<iterations>:<salt>$<stored_key>:<server_key>"
# Or for md5: "username" "md5<hash>"

# Generate password hash (from PostgreSQL):
# SELECT 'myuser' || ':' || passwd FROM pg_shadow WHERE usename = 'myuser';

# Simple approach: use plaintext for testing (not production!)
# "appuser" "password123"
# "pgbouncer_admin" "adminpassword"

# Better: use auth_query to pull credentials directly from PostgreSQL
```

## Using auth_query (Recommended)

```ini
# pgbouncer.ini
[pgbouncer]
auth_type = scram-sha-256
auth_user = pgbouncer_auth
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1
```

```sql
-- In PostgreSQL, create a limited auth user
CREATE ROLE pgbouncer_auth WITH LOGIN PASSWORD 'strongpassword';
GRANT pg_read_all_settings TO pgbouncer_auth;

-- Or minimal permission (PostgreSQL 14+):
GRANT EXECUTE ON FUNCTION pg_shadow_lookup TO pgbouncer_auth;
```

## Pool Size Tuning

```ini
[pgbouncer]
# Per-database pool size
default_pool_size = 25

# Minimum pool connections kept alive
min_pool_size = 5

# Extra connections to allow when pool is full (temporary overflow)
reserve_pool_size = 5
reserve_pool_timeout = 3.0

# Maximum client connections across all pools
max_client_conn = 1000

# Idle connection lifetime
server_idle_timeout = 600
client_idle_timeout = 0     # 0 = no timeout for clients

# Maximum connection age (force reconnect)
server_lifetime = 3600
```

## Firewall Configuration

```bash
# Allow application servers to reach PgBouncer
sudo iptables -A INPUT -s 10.0.0.0/24 -p tcp --dport 6432 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6432 -j DROP

# PgBouncer needs to reach PostgreSQL
sudo iptables -A OUTPUT -d 127.0.0.1 -p tcp --dport 5432 -j ACCEPT
```

## systemd Service

```ini
# /etc/systemd/system/pgbouncer.service (if not already present)
[Unit]
Description=PgBouncer connection pooler
After=network.target

[Service]
Type=forking
User=pgbouncer
ExecStart=/usr/sbin/pgbouncer -d /etc/pgbouncer/pgbouncer.ini
ExecReload=/bin/kill -HUP $MAINPID
PIDFile=/var/run/pgbouncer/pgbouncer.pid

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now pgbouncer
```

## Verifying the Configuration

```bash
# Check PgBouncer is listening on the right address
ss -tlnp | grep 6432
# Should show: 10.0.0.5:6432

# Connect to PgBouncer admin console
psql -h 10.0.0.5 -p 6432 -U pgbouncer_admin pgbouncer

# In admin console:
SHOW POOLS;
SHOW STATS;
SHOW DATABASES;
SHOW CLIENTS;
SHOW SERVERS;

# Connect through PgBouncer to the actual database
psql -h 10.0.0.5 -p 6432 -U appuser myapp
```

## Pooling Mode Comparison

| Mode | When to Use | Connection Reuse |
|---|---|---|
| `session` | Legacy apps using session state (temp tables, SET LOCAL) | One server connection per client session |
| `transaction` | Stateless web apps (most common) | Released after each transaction |
| `statement` | Simple single-statement apps | Released after each statement |

## Conclusion

PgBouncer listens on a specific IPv4 address by setting `listen_addr = 10.0.0.5` in `pgbouncer.ini`. Use `transaction` pool mode for most web applications - it provides the best connection reuse. Configure `auth_query` to pull credentials from PostgreSQL rather than maintaining a separate `userlist.txt`. Set `default_pool_size` based on your PostgreSQL `max_connections` limit (total across all PgBouncer instances must stay below that). Monitor pool health with `SHOW POOLS` and `SHOW STATS` in the admin console.
