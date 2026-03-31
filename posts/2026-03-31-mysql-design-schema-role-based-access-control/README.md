# How to Design a Schema for Role-Based Access Control in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, RBAC, Authorization, Schema Design

Description: Learn how to implement role-based access control (RBAC) in MySQL with roles, permissions, and user-role assignments with query examples.

---

Role-based access control (RBAC) assigns permissions to roles rather than directly to users. Users are then assigned one or more roles. This simplifies permission management as teams grow.

## Core Tables

```sql
CREATE TABLE users (
    id    INT UNSIGNED NOT NULL AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_email (email)
);

CREATE TABLE roles (
    id          INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name        VARCHAR(100) NOT NULL,
    description VARCHAR(255) NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_name (name)
);

CREATE TABLE permissions (
    id          INT UNSIGNED NOT NULL AUTO_INCREMENT,
    resource    VARCHAR(100) NOT NULL,
    action      VARCHAR(50)  NOT NULL,
    description VARCHAR(255) NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_resource_action (resource, action)
);
```

## Assignments

```sql
CREATE TABLE role_permissions (
    role_id       INT UNSIGNED NOT NULL,
    permission_id INT UNSIGNED NOT NULL,
    PRIMARY KEY (role_id, permission_id),
    KEY idx_permission (permission_id),
    CONSTRAINT fk_rp_role FOREIGN KEY (role_id)
        REFERENCES roles (id) ON DELETE CASCADE,
    CONSTRAINT fk_rp_permission FOREIGN KEY (permission_id)
        REFERENCES permissions (id) ON DELETE CASCADE
);

CREATE TABLE user_roles (
    user_id    INT UNSIGNED NOT NULL,
    role_id    INT UNSIGNED NOT NULL,
    granted_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    granted_by INT UNSIGNED NULL,
    PRIMARY KEY (user_id, role_id),
    KEY idx_role (role_id),
    CONSTRAINT fk_ur_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE,
    CONSTRAINT fk_ur_role FOREIGN KEY (role_id) REFERENCES roles (id) ON DELETE CASCADE
);
```

## Seeding Roles and Permissions

```sql
INSERT INTO roles (name, description) VALUES
    ('admin',  'Full system access'),
    ('editor', 'Can create and edit content'),
    ('viewer', 'Read-only access');

INSERT INTO permissions (resource, action) VALUES
    ('articles', 'read'),
    ('articles', 'create'),
    ('articles', 'update'),
    ('articles', 'delete'),
    ('users',    'read'),
    ('users',    'manage');

-- Assign permissions to the editor role
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id
FROM   roles r, permissions p
WHERE  r.name = 'editor'
  AND  p.resource = 'articles'
  AND  p.action IN ('read', 'create', 'update');
```

## Checking Permissions

```sql
-- Does user 42 have permission to delete articles?
SELECT COUNT(*) AS has_permission
FROM   user_roles ur
JOIN   role_permissions rp ON rp.role_id = ur.role_id
JOIN   permissions p ON p.id = rp.permission_id
WHERE  ur.user_id  = 42
  AND  p.resource  = 'articles'
  AND  p.action    = 'delete';
```

## All Permissions for a User

```sql
SELECT DISTINCT p.resource, p.action
FROM   permissions p
JOIN   role_permissions rp ON rp.permission_id = p.id
JOIN   user_roles ur       ON ur.role_id       = rp.role_id
WHERE  ur.user_id = 42
ORDER BY p.resource, p.action;
```

## Hierarchical Roles

For role inheritance, add a `parent_id` to `roles`:

```sql
ALTER TABLE roles ADD COLUMN parent_id INT UNSIGNED NULL;
ALTER TABLE roles ADD CONSTRAINT fk_role_parent FOREIGN KEY (parent_id) REFERENCES roles (id);
```

Then use a recursive CTE to resolve all inherited permissions at query time.

## Summary

RBAC schemas use three core tables - `users`, `roles`, and `permissions` - joined by `user_roles` and `role_permissions` junction tables. Check permissions with a three-table join filtered by resource and action. Seed roles and permissions on deploy. Add a `parent_id` to `roles` for hierarchical inheritance using recursive CTEs.
