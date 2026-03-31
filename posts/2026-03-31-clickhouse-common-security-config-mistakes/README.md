# Common ClickHouse Security Configuration Mistakes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Security, Access Control, TLS, Authentication

Description: Avoid the most common ClickHouse security configuration mistakes that leave clusters exposed to unauthorized access and data breaches.

---

ClickHouse is often deployed with default settings that prioritize convenience over security. These mistakes are common in production clusters that were set up quickly and never hardened.

## Mistake 1: Leaving the Default User Without a Password

The default user ships with no password and full access. Any process that can reach port 9000 or 8123 can read and delete all data.

```xml
<!-- users.xml: set a strong password for the default user -->
<users>
  <default>
    <password_sha256_hex>
      <!-- echo -n 'your_password' | sha256sum -->
      a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3
    </password_sha256_hex>
    <networks>
      <ip>127.0.0.1/32</ip>
    </networks>
  </default>
</users>
```

Restrict `<networks>` to localhost only and use a dedicated read-only user for application queries.

## Mistake 2: Not Enabling TLS on the HTTP and Native Interfaces

Without TLS, credentials and query results travel in plaintext over the network.

```xml
<!-- config.xml: enable HTTPS -->
<https_port>8443</https_port>
<tcp_port_secure>9440</tcp_port_secure>
<openSSL>
  <server>
    <certificateFile>/etc/clickhouse-server/server.crt</certificateFile>
    <privateKeyFile>/etc/clickhouse-server/server.key</privateKeyFile>
    <caConfig>/etc/ssl/certs/ca-certificates.crt</caConfig>
  </server>
</openSSL>
```

Disable the plain HTTP and TCP ports after confirming TLS works.

## Mistake 3: Granting Broad Permissions to Application Users

Application users should only have access to the tables and operations they need.

```sql
-- Create a restricted read-only user
CREATE USER app_reader IDENTIFIED BY 'secure_password';
GRANT SELECT ON analytics.* TO app_reader;

-- Create a write-only ingest user
CREATE USER ingest_writer IDENTIFIED BY 'another_password';
GRANT INSERT ON analytics.events TO ingest_writer;
```

Never grant `SYSTEM` or `DDL` privileges to application service accounts.

## Mistake 4: Exposing ClickHouse Ports Directly to the Internet

ClickHouse has no built-in rate limiting or IP throttling. Exposed ports are scanned and brute-forced constantly.

Use a firewall or security group to restrict access to trusted CIDR ranges. For public APIs, place an authenticated reverse proxy (nginx, Envoy) in front of the HTTP interface.

## Mistake 5: Not Enabling Interserver Authentication

In a cluster, ClickHouse nodes communicate via the interserver HTTP port (9009). Without authentication, any node on the network can join the cluster or read internal replication data.

```xml
<!-- config.xml -->
<interserver_http_credentials>
  <user>interserver</user>
  <password>strong_interserver_password</password>
</interserver_http_credentials>
```

Set the same credentials on all nodes in the cluster.

## Mistake 6: Leaving Query Log Accessible to All Users

`system.query_log` contains full query text including literal values. A read-only user can extract sensitive data from other users' queries.

```sql
REVOKE SELECT ON system.query_log FROM app_reader;
```

## Summary

Securing ClickHouse requires passwords on all users, TLS on all interfaces, least-privilege grants, firewall restrictions, interserver authentication, and restricted access to system tables. Apply these settings before exposing ClickHouse to any network that is not fully trusted.
