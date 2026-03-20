# How to Fix "no pg_hba.conf entry" and PostgreSQL IPv4 Connection Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: PostgreSQL, IPv4, Troubleshooting, pg_hba.conf, Connection Error, Database

Description: Diagnose and fix common PostgreSQL connection errors including "no pg_hba.conf entry for host", "password authentication failed", and connection timeout issues on IPv4.

## Introduction

PostgreSQL connection failures are usually one of three issues: the server isn't listening on the right IP, `pg_hba.conf` has no matching rule for the client, or the password/authentication method is wrong. This guide provides a systematic approach to resolving each.

## Common Errors and Root Causes

| Error Message | Root Cause |
|---|---|
| `FATAL: no pg_hba.conf entry for host "x.x.x.x"` | No matching rule in pg_hba.conf |
| `FATAL: password authentication failed` | Wrong password or authentication method mismatch |
| `could not connect to server: Connection refused` | PostgreSQL not listening on that IP/port |
| `could not connect to server: Connection timed out` | Firewall blocking the connection |
| `FATAL: role "username" does not exist` | User doesn't exist in PostgreSQL |

## Fix: "no pg_hba.conf entry"

```bash
# The client's IP is not in pg_hba.conf

# Check client's connecting IP from PostgreSQL log:
sudo grep "FATAL" /var/log/postgresql/postgresql-16-main.log | tail -5
# FATAL: no pg_hba.conf entry for host "10.0.0.50", user "appuser"...

# Add the required entry to pg_hba.conf:
sudo nano /etc/postgresql/16/main/pg_hba.conf

# Add line (before any reject rules):
# host  appdb  appuser  10.0.0.50/32  scram-sha-256

# Reload PostgreSQL (no restart needed for pg_hba.conf changes):
sudo systemctl reload postgresql

# Verify rules are loaded:
sudo -u postgres psql -c "SELECT * FROM pg_hba_file_rules;"
```

## Fix: Connection Refused

```bash
# Step 1: Is PostgreSQL running?
sudo systemctl status postgresql
sudo systemctl start postgresql

# Step 2: Check listen_addresses
sudo -u postgres psql -c "SHOW listen_addresses;"
# If 'localhost': not accepting remote connections

# Fix listen_addresses:
sudo nano /etc/postgresql/16/main/postgresql.conf
# listen_addresses = '*'

sudo systemctl restart postgresql

# Step 3: Verify listening
sudo ss -tlnp | grep 5432
```

## Fix: Connection Timeout

```bash
# Firewall is blocking port 5432

# Check UFW
sudo ufw status | grep 5432

# Check iptables
sudo iptables -L INPUT -n | grep 5432

# Add firewall rule
sudo ufw allow from 10.0.0.50 to any port 5432

# iptables
sudo iptables -A INPUT -p tcp --dport 5432 -s 10.0.0.50 -j ACCEPT

# Test with netcat
nc -zv 10.0.0.5 5432
```

## Fix: Authentication Method Mismatch

```bash
# Error: "FATAL: password authentication failed"
# Could be wrong password OR scram-sha-256 vs md5 mismatch

# Check the authentication method in pg_hba.conf:
sudo grep "appuser" /etc/postgresql/16/main/pg_hba.conf

# If pg_hba.conf says 'scram-sha-256' but user has MD5 password:
sudo -u postgres psql -e "ALTER USER appuser WITH ENCRYPTED PASSWORD 'newpassword';"

# Or change pg_hba.conf to 'md5' (less secure):
# host  all  all  10.0.0.0/24  md5
```

## Diagnostic One-Liner

```bash
# Quick health check for PostgreSQL remote access
HOST="10.0.0.5"
echo "=== Port Check ==="
nc -zv $HOST 5432
echo "=== PostgreSQL Status ==="
ssh $HOST "sudo systemctl status postgresql | head -5"
echo "=== Listening ==="
ssh $HOST "sudo ss -tlnp | grep 5432"
echo "=== pg_hba.conf entries ==="
ssh $HOST "sudo -u postgres psql -c 'SELECT * FROM pg_hba_file_rules;'"
```

## Conclusion

PostgreSQL connection errors follow a clear pattern: timeout = firewall, "Connection refused" = not listening, "no pg_hba.conf entry" = missing rule, "password authentication failed" = wrong credentials or method mismatch. Fix in order from network outward: firewall → binding → pg_hba.conf → credentials. Always reload (not restart) after pg_hba.conf changes.
