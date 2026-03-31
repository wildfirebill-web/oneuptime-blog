# How to Design a Multi-Tenant Database in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Multi-Tenant, Schema Design, SaaS

Description: Compare three multi-tenancy strategies in MySQL - shared schema, schema-per-tenant, and database-per-tenant - with practical implementation guidance.

---

Multi-tenancy means one application instance serves multiple customers (tenants) while keeping their data isolated. MySQL supports three main strategies, each with different trade-offs.

## Strategy 1 - Shared Schema With Tenant Column

All tenants share the same tables. Every table has a `tenant_id` column.

```sql
CREATE TABLE tenants (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
    slug VARCHAR(50)  NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_slug (slug)
);

CREATE TABLE projects (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    tenant_id  INT UNSIGNED NOT NULL,
    name       VARCHAR(255) NOT NULL,
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_tenant (tenant_id),
    CONSTRAINT fk_project_tenant FOREIGN KEY (tenant_id)
        REFERENCES tenants (id) ON DELETE CASCADE
);
```

Every query must include `WHERE tenant_id = ?`. Use row-level security in your application layer or views to enforce this.

```sql
-- Safe query pattern
SELECT id, name FROM projects
WHERE tenant_id = ? AND id = ?;
```

This strategy is the most resource-efficient but requires careful application-level isolation. A missing `tenant_id` filter leaks data across tenants.

## Strategy 2 - Schema Per Tenant

Each tenant gets its own schema (database in MySQL terminology):

```bash
# Create a schema for each new tenant
mysql -e "CREATE DATABASE tenant_acme;"
mysql -e "CREATE DATABASE tenant_globex;"
```

```sql
-- Run migrations per tenant schema
USE tenant_acme;
CREATE TABLE projects (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    PRIMARY KEY (id)
);
```

Isolation is stronger. Backups and migrations are per-tenant. The trade-off is managing hundreds or thousands of schemas.

## Strategy 3 - Database Per Tenant (Full Isolation)

Each tenant gets its own MySQL server or instance. Used when regulatory compliance demands complete isolation. Operationally the most expensive - typically reserved for enterprise customers.

## Recommended Hybrid Approach

Use shared schema for small tenants and promote to schema-per-tenant or database-per-tenant for large or regulated customers.

```sql
CREATE TABLE tenants (
    id               INT UNSIGNED NOT NULL AUTO_INCREMENT,
    slug             VARCHAR(50)  NOT NULL,
    isolation_level  ENUM('shared', 'schema', 'database') NOT NULL DEFAULT 'shared',
    database_name    VARCHAR(100) NULL,
    PRIMARY KEY (id)
);
```

## Handling Tenant Onboarding

```sql
-- Insert a new shared-schema tenant
INSERT INTO tenants (slug, isolation_level) VALUES ('acme-corp', 'shared');

-- Optionally run initial seed data in a transaction
START TRANSACTION;
SET @tid = LAST_INSERT_ID();
INSERT INTO projects (tenant_id, name) VALUES (@tid, 'Default Project');
COMMIT;
```

## Preventing Cross-Tenant Data Leaks

Create a view that automatically filters by tenant:

```sql
CREATE VIEW my_projects AS
SELECT * FROM projects
WHERE tenant_id = @current_tenant_id;
```

Set the session variable `@current_tenant_id` on connection to enforce tenant scope automatically.

## Summary

Shared schema is the most common and cost-effective multi-tenant approach in MySQL. Always index `tenant_id` and include it in every query. For sensitive tenants requiring stronger isolation, use schema-per-tenant. Use a `tenants` table with an `isolation_level` field to support a hybrid model that scales as your customer base grows.
