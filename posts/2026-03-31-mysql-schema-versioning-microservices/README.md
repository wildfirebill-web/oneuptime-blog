# How to Handle MySQL Schema Versioning in Microservices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Microservice, Schema, Migration, Versioning

Description: Learn how to manage MySQL schema versioning in a microservices architecture using migration tools, backward-compatible changes, and service-owned databases.

---

## Schema Versioning Challenges in Microservices

In a monolithic application, all code and the database schema evolve together. In microservices, each service owns its own database schema, and multiple service versions may run simultaneously during rolling deployments. This creates challenges:

- Service v1.2 and v1.1 may run at the same time during deployment
- A migration that drops a column v1.1 still reads will break the old pods
- Coordinating schema changes across service and database deployment order

## The Database-per-Service Pattern

Each microservice should own its own MySQL database or schema. Never share a database between services:

```sql
-- Service A owns its schema
CREATE DATABASE order_service CHARACTER SET utf8mb4;

-- Service B owns a separate schema
CREATE DATABASE inventory_service CHARACTER SET utf8mb4;
```

This decouples schema evolution between services and prevents cross-service coupling through shared tables.

## Using Flyway for Schema Version Tracking

Flyway tracks applied migrations in a `flyway_schema_history` table:

```text
migrations/
  V1__create_orders_table.sql
  V2__add_status_column.sql
  V3__add_index_on_user_id.sql
```

```sql
-- V1__create_orders_table.sql
CREATE TABLE orders (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id BIGINT UNSIGNED NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
) ENGINE=InnoDB;
```

Run migrations at service startup:

```java
@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        // Flyway runs migrations automatically before the app starts
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

Flyway ensures the schema matches the expected version for the running service version.

## Backward-Compatible Migration Patterns

During a rolling deployment, old and new service instances run simultaneously. Follow the expand-contract pattern:

**Step 1 - Expand (add, do not remove):**

```sql
-- V4__add_shipping_address_nullable.sql
-- New column is nullable so old service code still works
ALTER TABLE orders ADD COLUMN shipping_address VARCHAR(500) NULL;
```

Deploy new service code that reads and writes the new column. Old instances ignore it.

**Step 2 - Contract (remove old column once all instances are updated):**

```sql
-- V5__remove_old_address_column.sql
-- Only run after all instances run the new code
ALTER TABLE orders DROP COLUMN old_address;
```

Never drop a column in the same migration or deployment as the code that stops using it.

## Rename Pattern

Renaming a column requires three steps:

```sql
-- Step 1: Add new column
ALTER TABLE orders ADD COLUMN customer_id BIGINT UNSIGNED NULL;

-- Step 2: Backfill (in background or via migration)
UPDATE orders SET customer_id = user_id;

-- Step 3: Make NOT NULL and drop old column (after all code uses new name)
ALTER TABLE orders MODIFY customer_id BIGINT UNSIGNED NOT NULL;
ALTER TABLE orders DROP COLUMN user_id;
```

## Versioning Migrations with the Service Version

Name migrations to include the service version for traceability:

```text
V20250101_001__create_orders.sql
V20250115_001__add_status_column.sql
V20250130_001__add_shipping_address.sql
```

This makes it clear which service release introduced each schema change.

## Running Migrations in Kubernetes

In Kubernetes, run migrations as a Job before updating the Deployment:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: order-service-migrate-v3
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: order-service:v3.0.0
          command: ["flyway", "migrate"]
          env:
            - name: FLYWAY_URL
              value: jdbc:mysql://mysql.default.svc.cluster.local:3306/order_service
      restartPolicy: OnFailure
```

## Summary

Handle MySQL schema versioning in microservices by assigning each service its own database, using Flyway or Liquibase for version-tracked migrations, and following the expand-contract pattern for backward-compatible changes. Run migrations as a Kubernetes Job before rolling out new service Pods, and never drop or rename columns in the same deployment that removes the code using them.
