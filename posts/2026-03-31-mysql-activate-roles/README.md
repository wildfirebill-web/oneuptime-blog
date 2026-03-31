# How to Activate Roles in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Role, Security, User, Database Administration

Description: Learn how to activate MySQL roles in a session using SET ROLE, configure automatic activation with default roles, and check which roles are currently active.

---

## Why Roles Must Be Activated

In MySQL 8.0, when a role is granted to a user, it is not automatically active in new sessions. The user has the role but must activate it to gain its privileges. This two-step model - grant then activate - allows fine-grained control over when elevated privileges are available.

## Checking Currently Active Roles

```sql
SELECT CURRENT_ROLE();
```

Output when no roles are active:

```text
+----------------+
| CURRENT_ROLE() |
+----------------+
| NONE           |
+----------------+
```

## Activating a Specific Role

```sql
SET ROLE 'developer';
```

Verify:

```sql
SELECT CURRENT_ROLE();
```

```text
+-------------------+
| CURRENT_ROLE()    |
+-------------------+
| `developer`@`%`   |
+-------------------+
```

## Activating Multiple Roles

```sql
SET ROLE 'read_only', 'analyst';
```

## Activating All Granted Roles

```sql
SET ROLE ALL;
```

This activates every role currently granted to the user in this session.

## Deactivating All Roles

```sql
SET ROLE NONE;
```

After this, `CURRENT_ROLE()` returns `NONE` and the user has only their directly granted privileges (not role-based ones).

## Activating All Roles Except One

```sql
SET ROLE ALL EXCEPT 'superadmin';
```

Useful when a user has multiple roles and wants to avoid accidentally running commands with a high-privilege role.

## Default Roles - Automatic Activation

To avoid manually running `SET ROLE` on every login, set default roles:

```sql
SET DEFAULT ROLE 'developer' TO 'alice'@'localhost';
```

Now `developer` is automatically active when Alice logs in. Verify by logging in and checking:

```sql
SELECT CURRENT_ROLE();
```

## Server-Wide Automatic Activation

Enable automatic activation of all granted roles for all users at login:

```sql
SET GLOBAL activate_all_roles_on_login = ON;
```

Or persistently in `my.cnf`:

```text
[mysqld]
activate_all_roles_on_login = ON
```

## Checking Available (Granted but Possibly Not Active) Roles

```sql
SELECT ROLE_NAME, IS_DEFAULT, IS_MANDATORY
FROM information_schema.APPLICABLE_ROLES;
```

This shows all roles the current user has been granted, whether they are active by default, and whether they are mandatory.

## Practical Example - Privilege Escalation Pattern

Some teams configure users to log in with minimal privileges and only escalate when needed:

```sql
-- alice logs in with NONE active by default
SET DEFAULT ROLE NONE TO 'alice'@'localhost';

-- For sensitive operations, alice explicitly activates the needed role
SET ROLE 'admin_role';
-- ... perform admin tasks ...
SET ROLE NONE;  -- drop back to minimal privileges
```

This pattern reduces the risk of accidental data modification during normal operations.

## Summary

MySQL roles granted to a user are inactive by default. Use `SET ROLE 'rolename'` to activate specific roles in the current session, `SET ROLE ALL` to activate all of them, and `SET ROLE NONE` to deactivate. For persistent automatic activation, use `SET DEFAULT ROLE` per user or enable `activate_all_roles_on_login` globally. Always verify with `SELECT CURRENT_ROLE()` after activating.
