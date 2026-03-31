# How to Set Up Kerberos Authentication in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Kerberos, Authentication, Security, Enterprise

Description: Learn how to configure Kerberos authentication in ClickHouse to integrate with Kerberos KDC for secure single sign-on in enterprise environments.

---

## What Is Kerberos Authentication in ClickHouse

ClickHouse supports GSSAPI/Kerberos authentication, allowing users to authenticate using Kerberos tickets from a Key Distribution Center (KDC). This enables single sign-on (SSO) integration with corporate Active Directory or MIT Kerberos environments.

## Prerequisites

```bash
# Install Kerberos client libraries
# Debian/Ubuntu
apt-get install -y krb5-user libkrb5-dev

# RHEL/CentOS
yum install -y krb5-workstation krb5-libs
```

## Step 1 - Configure /etc/krb5.conf

```ini
[libdefaults]
    default_realm = COMPANY.COM
    dns_lookup_kdc = true
    dns_lookup_realm = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true

[realms]
    COMPANY.COM = {
        kdc = kdc1.company.com
        admin_server = kdc1.company.com
    }

[domain_realm]
    .company.com = COMPANY.COM
    company.com = COMPANY.COM
```

## Step 2 - Create a Service Principal and Keytab

On the KDC or Active Directory server:

```bash
# Create the ClickHouse service principal
kadmin.local -q "addprinc -randkey clickhouse/clickhouse-server.company.com@COMPANY.COM"

# Export the keytab
kadmin.local -q "ktadd -k /etc/clickhouse-server/clickhouse.keytab clickhouse/clickhouse-server.company.com@COMPANY.COM"

# Set permissions
chown clickhouse:clickhouse /etc/clickhouse-server/clickhouse.keytab
chmod 600 /etc/clickhouse-server/clickhouse.keytab
```

## Step 3 - Configure Kerberos in config.xml

```xml
<kerberos>
    <principal>clickhouse/clickhouse-server.company.com@COMPANY.COM</principal>
    <keytab>/etc/clickhouse-server/clickhouse.keytab</keytab>
    <realm>COMPANY.COM</realm>
</kerberos>
```

## Step 4 - Configure User Authentication in users.xml

```xml
<users>
    <alice>
        <kerberos>
            <realm>COMPANY.COM</realm>
        </kerberos>
        <networks>
            <ip>::/0</ip>
        </networks>
        <profile>default</profile>
        <quota>default</quota>
    </alice>
</users>
```

## Step 5 - Configure an External User Directory for GSSAPI

To allow any valid Kerberos principal to authenticate:

```xml
<user_directories>
    <users_xml>
        <path>users.xml</path>
    </users_xml>
    <ldap>
        <server>corporate_ldap</server>
        <roles>
            <default/>
        </roles>
    </ldap>
</user_directories>
```

## Step 6 - Authenticate with Kerberos from clickhouse-client

```bash
# Obtain a Kerberos ticket
kinit alice@COMPANY.COM

# Verify ticket
klist

# Connect to ClickHouse using GSSAPI
clickhouse-client \
    --host clickhouse-server.company.com \
    --secure \
    --gssapi-username alice
```

## Verifying Keytab

```bash
# Test the keytab
kinit -kt /etc/clickhouse-server/clickhouse.keytab \
    clickhouse/clickhouse-server.company.com@COMPANY.COM

klist
```

## Configuring ClickHouse JDBC Driver for Kerberos

```java
// JDBC URL with Kerberos
String url = "jdbc:ch://clickhouse-server.company.com:8123/default?" +
             "ssl=true&" +
             "security_token=kerberos&" +
             "principal=clickhouse/clickhouse-server.company.com@COMPANY.COM";

// Set Kerberos system properties
System.setProperty("java.security.krb5.conf", "/etc/krb5.conf");
System.setProperty("javax.security.auth.useSubjectCredsOnly", "false");

Connection conn = DriverManager.getConnection(url, "", "");
```

## Monitoring Kerberos Authentication

```sql
-- Check Kerberos login activity
SELECT
    event_time,
    user,
    client_hostname,
    type,
    message
FROM system.session_log
WHERE event_date = today()
  AND message LIKE '%Kerberos%'
ORDER BY event_time DESC
LIMIT 20;
```

## Troubleshooting Common Errors

```bash
# Clock skew error - sync time
ntpdate -u kdc1.company.com

# Keytab not readable
chmod 600 /etc/clickhouse-server/clickhouse.keytab
chown clickhouse:clickhouse /etc/clickhouse-server/clickhouse.keytab

# Principal not found - verify the principal
kinit -kt /etc/clickhouse-server/clickhouse.keytab \
    clickhouse/clickhouse-server.company.com@COMPANY.COM

# Debug GSSAPI
export KRB5_TRACE=/tmp/krb5_trace.log
```

## Summary

Kerberos authentication in ClickHouse uses GSSAPI to validate user tickets against a KDC. The setup requires creating a service principal and keytab, configuring `/etc/krb5.conf`, enabling Kerberos in `config.xml`, and defining users with `<kerberos><realm>` in `users.xml`. Users authenticate by running `kinit` and then connecting via `clickhouse-client --gssapi-username`. Keep the keytab file secured with restricted permissions and ensure clock synchronization with the KDC.
