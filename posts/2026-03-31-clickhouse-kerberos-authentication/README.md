# How to Configure Kerberos Authentication in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Kerberos, Security, Authentication, Database, Configuration

Description: Learn how to set up Kerberos authentication in ClickHouse so users and services can authenticate using Kerberos tickets from an enterprise KDC or Active Directory.

---

Kerberos is an enterprise authentication protocol that uses cryptographic tickets issued by a trusted Key Distribution Center (KDC). Organizations that already use Kerberos (typically through Active Directory or MIT Kerberos) can configure ClickHouse to accept Kerberos tickets as the authentication mechanism. This removes the need to manage separate ClickHouse passwords for each user.

## How Kerberos Authentication Works in ClickHouse

ClickHouse uses the Generic Security Services API (GSS-API) via GSSAPI to authenticate incoming connections. The flow is:

1. The client (clickhouse-client, JDBC driver, or application) obtains a Kerberos service ticket for the ClickHouse server principal
2. The ticket is presented to ClickHouse during the connection handshake
3. ClickHouse verifies the ticket using its keytab file
4. If valid, the user is authenticated; ClickHouse applies local privileges

Kerberos authentication in ClickHouse applies to the native TCP protocol and HTTP interface.

## Prerequisites

- A running Kerberos KDC (MIT Kerberos or Active Directory)
- A service principal for ClickHouse in the KDC
- A keytab file exported from the KDC placed on the ClickHouse server
- MIT Kerberos libraries installed on the ClickHouse host

## Installing Kerberos Client Libraries

```bash
# Debian/Ubuntu
apt-get install -y krb5-user libkrb5-dev

# RHEL/CentOS
yum install -y krb5-workstation krb5-libs
```

## Creating the ClickHouse Service Principal

On the KDC or via Active Directory:

```bash
# MIT Kerberos: create service principal
kadmin.local -q "addprinc -randkey HTTP/clickhouse.corp.example.com@CORP.EXAMPLE.COM"
kadmin.local -q "addprinc -randkey clickhouse/clickhouse.corp.example.com@CORP.EXAMPLE.COM"

# Export to keytab
kadmin.local -q "ktadd -k /etc/clickhouse-server/clickhouse.keytab HTTP/clickhouse.corp.example.com@CORP.EXAMPLE.COM"
kadmin.local -q "ktadd -k /etc/clickhouse-server/clickhouse.keytab clickhouse/clickhouse.corp.example.com@CORP.EXAMPLE.COM"
```

Set ownership so ClickHouse can read the keytab:

```bash
chown clickhouse:clickhouse /etc/clickhouse-server/clickhouse.keytab
chmod 600 /etc/clickhouse-server/clickhouse.keytab
```

## Configuring /etc/krb5.conf

The Kerberos client library on the ClickHouse host needs to know where the KDC is:

```text
[libdefaults]
    default_realm = CORP.EXAMPLE.COM
    dns_lookup_realm = false
    dns_lookup_kdc = true
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true

[realms]
    CORP.EXAMPLE.COM = {
        kdc = kdc1.corp.example.com
        kdc = kdc2.corp.example.com
        admin_server = kdc1.corp.example.com
    }

[domain_realm]
    .corp.example.com = CORP.EXAMPLE.COM
    corp.example.com = CORP.EXAMPLE.COM
```

## Configuring Kerberos in ClickHouse

Add the Kerberos configuration to `/etc/clickhouse-server/config.d/kerberos.xml`:

```xml
<clickhouse>
  <kerberos>
    <!-- Kerberos realm (must match KDC configuration) -->
    <realm>CORP.EXAMPLE.COM</realm>
    <!-- Path to the server's keytab file -->
    <keytab>/etc/clickhouse-server/clickhouse.keytab</keytab>
    <!-- Service principal name -->
    <principal>clickhouse/clickhouse.corp.example.com@CORP.EXAMPLE.COM</principal>
  </kerberos>
</clickhouse>
```

## Creating Kerberos-Authenticated Users

Create users in ClickHouse that authenticate via Kerberos. The username must match the Kerberos principal name (without the realm):

```sql
-- User 'alice' authenticated via Kerberos realm CORP.EXAMPLE.COM
CREATE USER alice
    IDENTIFIED WITH kerberos REALM 'CORP.EXAMPLE.COM';

-- Grant privileges
GRANT SELECT ON analytics.* TO alice;
GRANT SELECT ON metrics.* TO alice;
```

To accept users from any realm (useful when multiple realms are federated):

```sql
CREATE USER svc_etl
    IDENTIFIED WITH kerberos;
```

## Defining Kerberos Users in users.xml

Alternatively, define Kerberos users in `/etc/clickhouse-server/users.xml`:

```xml
<users>
  <alice>
    <kerberos>
      <realm>CORP.EXAMPLE.COM</realm>
    </kerberos>
    <networks>
      <ip>::/0</ip>
    </networks>
    <profile>default</profile>
    <quota>default</quota>
  </alice>
</users>
```

## Connecting with clickhouse-client Using Kerberos

First, obtain a Kerberos ticket on the client machine:

```bash
kinit alice@CORP.EXAMPLE.COM
```

Then connect to ClickHouse:

```bash
clickhouse-client \
  --host clickhouse.corp.example.com \
  --user alice \
  --password ""
```

ClickHouse will use the existing Kerberos credentials cached by `kinit` for authentication.

## Connecting via HTTP with Kerberos (SPNEGO)

For HTTP connections (Grafana, BI tools), ClickHouse supports SPNEGO (HTTP Negotiate authentication). Configure the HTTP interface in `config.xml`:

```xml
<clickhouse>
  <http_server_default_response><![CDATA[<html><body>It works!</body></html>]]></http_server_default_response>
  <!-- Enable SPNEGO for HTTP -->
  <kerberos>
    <realm>CORP.EXAMPLE.COM</realm>
    <keytab>/etc/clickhouse-server/clickhouse.keytab</keytab>
    <principal>HTTP/clickhouse.corp.example.com@CORP.EXAMPLE.COM</principal>
  </kerberos>
</clickhouse>
```

Use curl to test SPNEGO:

```bash
curl --negotiate -u : http://clickhouse.corp.example.com:8123/?query=SELECT+1
```

## Verifying the Keytab

Test that the keytab is valid and the service principal exists:

```bash
kinit -kt /etc/clickhouse-server/clickhouse.keytab \
  clickhouse/clickhouse.corp.example.com@CORP.EXAMPLE.COM

klist
```

## Checking Active Kerberos Sessions

```sql
SELECT user, client_hostname, query_start_time, auth_type
FROM system.processes
WHERE user = 'alice';
```

Verify authentication method in session log:

```sql
SELECT
    user,
    client_hostname,
    auth_type,
    event_time
FROM system.session_log
WHERE auth_type = 'Kerberos'
ORDER BY event_time DESC
LIMIT 20;
```

## Troubleshooting

**KDC unreachable:**

```bash
telnet kdc1.corp.example.com 88
```

**Clock skew error:**
Kerberos requires clocks to be within 5 minutes of each other. Sync with NTP:

```bash
timedatectl set-ntp true
```

**Keytab permission denied:**

```bash
sudo -u clickhouse klist -kt /etc/clickhouse-server/clickhouse.keytab
```

## Summary

Kerberos authentication in ClickHouse allows users to authenticate with existing enterprise Kerberos tickets. Create a service principal and keytab, configure the realm in ClickHouse's XML config, and create users with `IDENTIFIED WITH kerberos`. For HTTP access, ClickHouse supports SPNEGO. Authorization remains managed by ClickHouse's own privilege system independent of Kerberos.
