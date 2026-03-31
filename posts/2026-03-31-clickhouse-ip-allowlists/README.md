# How to Configure IP Allowlists in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Security, Network, Database, Administration, Configuration

Description: Learn how to restrict ClickHouse user access by IP address and CIDR range using host-level restrictions in user definitions and server-level network configuration.

---

IP allowlisting in ClickHouse restricts which network addresses can authenticate as a given user. Even if a user has the correct password or certificate, a connection from an IP address not in their allowlist is rejected before authentication is attempted. This is a critical defense-in-depth measure that limits the blast radius if credentials are compromised.

## How IP Restrictions Work

IP restrictions in ClickHouse are defined per user. They apply to both the native TCP interface (port 9000/9440) and the HTTP interface (port 8123/8443). When a connection arrives, ClickHouse checks the client's IP against the user's allowed network list before proceeding with authentication.

## Configuring IP Restrictions via SQL

When creating a user with SQL, use the `HOST` clause:

```sql
-- Allow from a single IP address
CREATE USER alice
    IDENTIFIED WITH sha256_password BY 'AlicePass123!'
    HOST IP '192.168.1.50';

-- Allow from a CIDR range (entire /24 subnet)
CREATE USER api_service
    IDENTIFIED WITH sha256_password BY 'ApiPass456!'
    HOST IP '10.10.5.0/24';

-- Allow from multiple specific IPs
CREATE USER analyst
    IDENTIFIED WITH sha256_password BY 'AnalystPass789!'
    HOST IP '172.16.0.10',
    HOST IP '172.16.0.11',
    HOST IP '172.16.0.12';

-- Allow from multiple CIDR ranges
CREATE USER etl_pipeline
    IDENTIFIED WITH sha256_password BY 'EtlPass000!'
    HOST IP '10.0.0.0/8',
    HOST IP '192.168.100.0/24';

-- Allow from localhost only
CREATE USER local_admin
    IDENTIFIED WITH sha256_password BY 'LocalPass111!'
    HOST LOCAL;

-- Allow from any host (use only for dev or trusted internal users)
CREATE USER dev_user
    IDENTIFIED WITH sha256_password BY 'DevPass222!'
    HOST ANY;
```

## Updating IP Restrictions on an Existing User

```sql
-- Replace the host restrictions entirely
ALTER USER alice
    HOST IP '10.0.0.0/8';

-- Add IPv6 loopback
ALTER USER local_admin
    HOST LOCAL,
    HOST IP '::1';
```

## Combining IP Restrictions with Hostname Restrictions

ClickHouse also supports hostname-based restrictions. The hostname is resolved at connection time:

```sql
CREATE USER external_reporter
    IDENTIFIED WITH sha256_password BY 'ReporterPass333!'
    HOST NAME 'reporting-server.corp.example.com',
    HOST NAME 'bi-tool.corp.example.com';
```

You can mix IP and hostname restrictions in the same user definition:

```sql
CREATE USER grafana
    IDENTIFIED WITH sha256_password BY 'GrafanaPass444!'
    HOST IP '10.0.0.5',
    HOST NAME 'grafana.internal';
```

## Configuring IP Restrictions in users.xml

For users defined in `users.xml` (the legacy approach):

```xml
<users>
  <alice>
    <password_sha256_hex>8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918</password_sha256_hex>
    <networks>
      <!-- Single IP -->
      <ip>192.168.1.50</ip>
      <!-- CIDR range -->
      <ip>10.10.5.0/24</ip>
      <!-- IPv6 range -->
      <ip>fd00::/8</ip>
    </networks>
    <profile>default</profile>
    <quota>default</quota>
  </alice>

  <local_only_user>
    <password_sha256_hex>...</password_sha256_hex>
    <networks>
      <ip>127.0.0.1</ip>
      <ip>::1</ip>
    </networks>
    <profile>default</profile>
    <quota>default</quota>
  </local_only_user>
</users>
```

## Server-Level Network Restrictions

In addition to per-user IP restrictions, you can restrict which interfaces ClickHouse listens on at the server level. Edit `config.xml`:

```xml
<clickhouse>
  <!-- Bind to a specific interface only (not all interfaces) -->
  <listen_host>10.0.0.50</listen_host>
  <!-- Allow localhost as well -->
  <listen_host>127.0.0.1</listen_host>
  <listen_host>::1</listen_host>
</clickhouse>
```

This prevents connections from reaching ClickHouse at all from unlisted interfaces, before any user-level checks occur.

## Blocking External Access with a Firewall

Supplement ClickHouse IP restrictions with OS-level firewall rules. Using `ufw` on Ubuntu:

```bash
# Allow ClickHouse native port only from trusted subnet
ufw allow from 10.0.0.0/8 to any port 9000 proto tcp
ufw allow from 10.0.0.0/8 to any port 9440 proto tcp

# Allow HTTP interface only from internal network
ufw allow from 10.0.0.0/8 to any port 8123 proto tcp
ufw allow from 10.0.0.0/8 to any port 8443 proto tcp

# Block all other access to ClickHouse ports
ufw deny 9000
ufw deny 9440
ufw deny 8123
ufw deny 8443
```

Using `iptables`:

```bash
# Allow native protocol from trusted CIDR
iptables -A INPUT -p tcp --dport 9000 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 9000 -j DROP

# Allow HTTP interface from trusted CIDR
iptables -A INPUT -p tcp --dport 8123 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 8123 -j DROP
```

## Viewing Current IP Restrictions

```sql
-- Check all users and their host restrictions
SELECT
    name,
    host_ip,
    host_names,
    host_names_regexp,
    host_names_like
FROM system.users
ORDER BY name;
```

## Testing IP Restriction Behavior

From a permitted IP, the connection succeeds:

```bash
clickhouse-client --host 10.0.0.50 --user alice --password AlicePass123!
```

From a blocked IP, ClickHouse returns:

```text
DB::Exception: alice: Authentication failed: password is incorrect, or there is no user with such name.
```

Note that ClickHouse returns the same error for both wrong passwords and IP blocks to prevent user enumeration.

## Allowing Specific IPs for Monitoring Users

Create a read-only monitoring user restricted to your monitoring server's IP:

```sql
CREATE USER prometheus_reader
    IDENTIFIED WITH sha256_password BY 'MonitorPass555!'
    HOST IP '10.0.1.20';

GRANT SELECT ON system.* TO prometheus_reader;
GRANT SELECT ON system.metrics TO prometheus_reader;
GRANT SELECT ON system.asynchronous_metrics TO prometheus_reader;
```

## Summary

ClickHouse IP allowlists are configured per user through the `HOST IP` clause in SQL or the `<networks>` block in `users.xml`. Restrict each user to the minimum set of IP addresses needed for their function. Supplement user-level restrictions with server `listen_host` configuration to bind ClickHouse to internal interfaces only, and add OS-level firewall rules as a final layer of defense. Use `system.users` to audit current host restrictions.
