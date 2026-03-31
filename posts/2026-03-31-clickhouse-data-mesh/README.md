# How to Build a Data Mesh with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Mesh, Data Architecture, Domain-Driven Design, Federation

Description: Build a data mesh with ClickHouse by assigning domain ownership to separate databases and using federated queries to compose cross-domain analytics.

---

## Data Mesh Principles

A data mesh decentralizes data ownership. Each domain team owns their data products - schemas, pipelines, and SLAs. A central platform team provides infrastructure (like ClickHouse) but not data itself. Data products are discoverable and accessible by other domains through agreed interfaces.

## Domain Isolation in ClickHouse

Map each domain to its own ClickHouse database. This enforces ownership boundaries:

```sql
CREATE DATABASE ecommerce_orders;
CREATE DATABASE ecommerce_users;
CREATE DATABASE marketing_campaigns;
CREATE DATABASE finance_revenue;
```

Domain teams manage their own tables and materialized views within their databases.

## Data Product Contracts via Views

Each domain publishes a stable data product as a view. Other domains query this view, not the underlying tables:

```sql
-- Published by the Orders domain team
CREATE VIEW ecommerce_orders.orders_product AS
SELECT
    order_id,
    user_id,
    toDate(created_at) AS order_date,
    status,
    total_amount_usd
FROM ecommerce_orders.orders_raw
WHERE status != 'test';
```

Internal table details can change; the view contract stays stable.

## Cross-Domain Federated Queries

ClickHouse supports querying across databases natively:

```sql
SELECT
    u.country,
    count(DISTINCT o.user_id) AS buyers,
    sum(o.total_amount_usd) AS revenue
FROM ecommerce_orders.orders_product o
JOIN ecommerce_users.users_product u ON u.user_id = o.user_id
WHERE o.order_date >= today() - 30
GROUP BY u.country
ORDER BY revenue DESC
LIMIT 20;
```

## Access Control for Domain Boundaries

Create roles aligned to domains and grant access only to the published views:

```sql
CREATE ROLE marketing_reader;
GRANT SELECT ON ecommerce_orders.orders_product TO marketing_reader;
GRANT SELECT ON ecommerce_users.users_product TO marketing_reader;

CREATE USER campaign_analyst
    IDENTIFIED WITH sha256_password BY 'secure_pass'
    DEFAULT ROLE marketing_reader;
```

Marketing analysts cannot access `ecommerce_orders.orders_raw` - only the public contract view.

## Data Product Catalog

Track data products in a ClickHouse table itself:

```sql
CREATE TABLE platform.data_catalog (
    domain          String,
    product_name    String,
    description     String,
    owner_team      String,
    sla_freshness   String,
    published_at    DateTime
) ENGINE = ReplacingMergeTree()
ORDER BY (domain, product_name);
```

Domain teams register their products here, making them discoverable.

## Measuring Data Product Health

Monitor freshness SLAs for each data product:

```sql
SELECT
    c.domain,
    c.product_name,
    c.sla_freshness,
    max(updated_at) AS last_updated,
    dateDiff('minute', max(updated_at), now()) AS lag_minutes
FROM platform.data_catalog c
JOIN platform.data_product_heartbeats h ON h.product_name = c.product_name
GROUP BY c.domain, c.product_name, c.sla_freshness
HAVING lag_minutes > 60
ORDER BY lag_minutes DESC;
```

## Summary

ClickHouse supports the data mesh pattern through database-per-domain isolation, view-based data product contracts, native cross-database queries, and fine-grained RBAC - giving teams ownership while preserving the ability to compose cross-domain analytics.
