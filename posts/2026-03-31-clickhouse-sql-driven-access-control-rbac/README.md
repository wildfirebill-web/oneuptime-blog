# How to Use SQL-Driven Access Control (RBAC) in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, RBAC, Access Control, Security, Role, Privilege

Description: Learn how to implement SQL-driven Role-Based Access Control in ClickHouse to manage users, roles, and privileges with standard SQL commands.

---

ClickHouse supports SQL-driven access control (also called RBAC) starting from version 20.4. This replaces the older `users.xml` configuration with a more flexible, dynamic approach managed entirely through SQL statements.

## Enabling SQL-Driven Access Control

First, enable the access control directory in `config.xml`:

```text
<access_control_path>/var/lib/clickhouse/access/</access_control_path>
```

Restart ClickHouse. The system is now ready for SQL-managed users and roles.

## Creating Users

```sql
CREATE USER analyst
IDENTIFIED WITH sha256_password BY 'AnalystPass123!'
DEFAULT DATABASE analytics_db;
```

Verify the user was created:

```sql
SELECT name, auth_type, host_ip, default_database
FROM system.users
WHERE name = 'analyst';
```

## Creating Roles

Roles group privileges and can be assigned to multiple users:

```sql
CREATE ROLE data_reader;
CREATE ROLE data_writer;
CREATE ROLE db_admin;
```

## Granting Privileges to Roles

```sql
-- Read-only role
GRANT SELECT ON analytics_db.* TO data_reader;

-- Read-write role
GRANT SELECT, INSERT ON analytics_db.* TO data_writer;

-- Admin role
GRANT ALL ON analytics_db.* TO db_admin;
```

Grant privileges at the column level for fine-grained control:

```sql
GRANT SELECT(user_id, event_type, timestamp) ON analytics_db.events TO data_reader;
```

## Assigning Roles to Users

```sql
GRANT data_reader TO analyst;
```

Set a default role that activates at login:

```sql
ALTER USER analyst DEFAULT ROLE data_reader;
```

Users can also activate roles manually in a session:

```sql
SET ROLE data_writer;
```

## Viewing Grants and Roles

```sql
-- See all grants for a user
SHOW GRANTS FOR analyst;

-- See all roles
SHOW ROLES;

-- Check current user's grants
SHOW GRANTS;
```

## Role Hierarchy

Roles can be granted to other roles, creating hierarchies:

```sql
CREATE ROLE senior_analyst;
GRANT data_reader TO senior_analyst;
GRANT SHOW DATABASES ON *.* TO senior_analyst;
GRANT senior_analyst TO senior_user;
```

## Revoking Privileges

```sql
-- Revoke specific privilege
REVOKE INSERT ON analytics_db.* FROM data_writer;

-- Revoke a role from a user
REVOKE data_reader FROM analyst;

-- Drop a role entirely
DROP ROLE data_reader;
```

## Using GRANT OPTION

Allow a user to further grant their permissions to others:

```sql
GRANT SELECT ON analytics_db.events TO team_lead WITH GRANT OPTION;
```

## Checking Access in Practice

```sql
-- Check if current user can perform an action
SELECT * FROM system.grants WHERE user_name = 'analyst';
```

## Best Practices

- Follow the principle of least privilege - grant only what is needed
- Use roles rather than granting privileges directly to users
- Separate read and write roles for different teams
- Audit grants regularly using `system.grants` and `system.role_grants`
- Use `DEFAULT DATABASE` when creating users to limit scope

```sql
SELECT user_name, role_name, access_type, database, table
FROM system.grants
ORDER BY user_name, database;
```

## Summary

SQL-driven RBAC in ClickHouse provides a flexible and auditable approach to access control. By creating roles with appropriate privileges and assigning them to users, you can enforce least-privilege access across your entire ClickHouse deployment without editing XML files.
