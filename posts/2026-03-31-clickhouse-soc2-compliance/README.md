# How to Configure ClickHouse for SOC 2 Compliance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SOC 2, Compliance, Security, Audit, Access Control

Description: Learn the key ClickHouse configuration steps to support SOC 2 Type II compliance including access controls, audit logging, encryption, and monitoring.

---

SOC 2 compliance requires demonstrating that your systems protect data through security, availability, and confidentiality controls. ClickHouse can be configured to satisfy the technical requirements of SOC 2 Type II audits.

## SOC 2 Trust Service Criteria Relevant to ClickHouse

The five Trust Service Criteria (TSC) map to ClickHouse controls:

| TSC | ClickHouse Control |
| --- | --- |
| CC6 - Logical Access | RBAC, user authentication, IP whitelisting |
| CC7 - System Operations | Query logging, monitoring |
| CC8 - Change Management | Config version control, audit logs |
| A1 - Availability | Replication, backups |
| C1 - Confidentiality | Encryption at rest, TLS in transit |

## Access Control (CC6)

Implement least-privilege access with SQL RBAC:

```sql
-- Create role-based access structure
CREATE ROLE readonly_analyst;
CREATE ROLE data_engineer;
CREATE ROLE dba;

GRANT SELECT ON production_db.* TO readonly_analyst;
GRANT SELECT, INSERT ON staging_db.* TO data_engineer;
GRANT ALL ON production_db.* TO dba;

-- All users must authenticate with strong passwords
CREATE USER alice
IDENTIFIED WITH sha256_password BY 'ComplexPass123!'
HOST IP '10.0.0.0/24';

GRANT readonly_analyst TO alice;
```

## Audit Logging (CC7)

Enable comprehensive query logging:

```text
<query_log>
  <database>system</database>
  <table>query_log</table>
  <flush_interval_milliseconds>7500</flush_interval_milliseconds>
</query_log>
```

Query the audit trail:

```sql
SELECT
    event_time,
    user,
    client_hostname,
    query_kind,
    databases,
    tables,
    query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date >= today() - 30
ORDER BY event_time DESC;
```

## Encryption at Rest (C1)

Enable disk-level encryption in `config.xml`:

```text
<storage_configuration>
  <disks>
    <encrypted_disk>
      <type>encrypted</type>
      <disk>default</disk>
      <path>encrypted/</path>
      <algorithm>AES_128_CTR</algorithm>
      <key_hex>your_32_char_hex_key_here</key_hex>
    </encrypted_disk>
  </disks>
</storage_configuration>
```

## Encryption in Transit (C1)

Enforce TLS for all connections:

```text
<https_port>8443</https_port>
<tcp_port_secure>9440</tcp_port_secure>
<!-- Comment out plaintext ports -->
<!-- <http_port>8123</http_port> -->
<!-- <tcp_port>9000</tcp_port> -->
```

## Password Policy (CC6)

Enforce password complexity and expiry through user management procedures. Document these in your security runbooks as ClickHouse does not enforce password complexity natively - use SHA256 or bcrypt hashing at minimum:

```sql
-- Verify password hashing method for all users
SELECT name, auth_type
FROM system.users
WHERE auth_type NOT IN ('sha256_password', 'double_sha1_password', 'bcrypt_password');
```

## Session Controls (CC6)

Set session-level timeouts:

```text
<profiles>
  <default>
    <max_execution_time>300</max_execution_time>
    <readonly>0</readonly>
    <max_memory_usage>10000000000</max_memory_usage>
  </default>
</profiles>
```

## Access Reviews

Generate access review reports for auditors:

```sql
-- All users and their roles
SELECT u.name AS user_name, r.granted_role_name AS role
FROM system.users u
LEFT JOIN system.role_grants r ON u.name = r.user_name
ORDER BY u.name;

-- All privileges granted to roles
SELECT role_name, access_type, database, table
FROM system.grants
WHERE role_name IS NOT NULL
ORDER BY role_name;
```

## Backup Controls (A1)

Document and automate regular backups:

```sql
-- Create a backup (ClickHouse 22.4+)
BACKUP DATABASE production_db TO S3('s3://backups/clickhouse/', 'key', 'secret');

-- Verify backup integrity
RESTORE DATABASE production_db FROM S3('s3://backups/clickhouse/', 'key', 'secret')
SETTINGS allow_non_empty_tables = false;
```

## Summary

SOC 2 compliance for ClickHouse involves configuring SQL RBAC for least-privilege access, enabling comprehensive query audit logging, enforcing TLS encryption in transit, configuring disk encryption at rest, and maintaining documented access review and backup procedures. These controls provide the evidence auditors need to verify your security posture.
