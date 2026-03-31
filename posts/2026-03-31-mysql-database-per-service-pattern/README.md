# How to Implement Database per Service Pattern with MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database Per Service, Microservice, Pattern

Description: Learn how to implement the database-per-service pattern with MySQL, including separate databases per service, connection isolation, and cross-service data access strategies.

---

## What Is Database per Service

The database-per-service pattern is a microservices design principle where each service has exclusive ownership of its own database. No other service can access that database directly. This gives each service full autonomy to choose its schema, evolve it independently, and scale its database without coordinating with other teams.

With MySQL, this typically means one MySQL database (schema) per service, possibly on separate MySQL instances for full isolation.

## Setting Up Separate Databases

```sql
-- Create isolated databases for each service
CREATE DATABASE order_svc   CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE payment_svc CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE product_svc CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE user_svc    CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Create dedicated MySQL users with access only to their own database:

```sql
-- order-service user can only access order_svc
CREATE USER 'order_app'@'%' IDENTIFIED BY 'strong_password_here';
GRANT SELECT, INSERT, UPDATE, DELETE ON order_svc.* TO 'order_app'@'%';

-- payment-service user can only access payment_svc
CREATE USER 'payment_app'@'%' IDENTIFIED BY 'strong_password_here';
GRANT SELECT, INSERT, UPDATE, DELETE ON payment_svc.* TO 'payment_app'@'%';
```

## Order Service Schema

```sql
USE order_svc;

CREATE TABLE orders (
  id           BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  order_uuid   CHAR(36)      NOT NULL DEFAULT (UUID()),
  user_uuid    CHAR(36)      NOT NULL,
  status       VARCHAR(20)   NOT NULL DEFAULT 'PENDING',
  total_amount DECIMAL(12,2) NOT NULL,
  created_at   TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uq_uuid  (order_uuid),
  INDEX idx_user      (user_uuid),
  INDEX idx_status    (status)
) ENGINE=InnoDB;

CREATE TABLE order_lines (
  id           BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  order_id     BIGINT        NOT NULL,
  product_uuid CHAR(36)      NOT NULL,
  product_name VARCHAR(200)  NOT NULL,  -- snapshot at order time
  unit_price   DECIMAL(10,2) NOT NULL,
  quantity     INT           NOT NULL,
  CONSTRAINT fk_line_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
) ENGINE=InnoDB;
```

## Payment Service Schema

```sql
USE payment_svc;

CREATE TABLE payments (
  id           BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  payment_uuid CHAR(36)      NOT NULL DEFAULT (UUID()),
  order_uuid   CHAR(36)      NOT NULL,  -- cross-service reference by UUID
  amount       DECIMAL(12,2) NOT NULL,
  currency     CHAR(3)       NOT NULL DEFAULT 'USD',
  status       VARCHAR(20)   NOT NULL DEFAULT 'INITIATED',
  provider     VARCHAR(50)   NOT NULL,
  processed_at TIMESTAMP,
  created_at   TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uq_uuid       (payment_uuid),
  UNIQUE KEY uq_order_uuid (order_uuid)
) ENGINE=InnoDB;
```

## Cross-Service Data Access via API

Services must never query each other's databases directly. Use HTTP/gRPC APIs:

```python
# order-service: when building an order receipt, fetch user details via API
import httpx

async def get_order_receipt(order_id: int, db_conn):
    # Query own database
    order = await db_conn.fetch_one(
        "SELECT id, order_uuid, user_uuid, total_amount FROM orders WHERE id = %s",
        (order_id,)
    )

    # Fetch user details from user-service API (not direct DB access)
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            f"http://user-service/internal/users/{order['user_uuid']}"
        )
        user = resp.json()

    return {"order": dict(order), "user": user}
```

## Connection Configuration per Service

Each service has its own connection pool pointing to its own database:

```yaml
# order-service application config
database:
  host: mysql-order.internal
  port: 3306
  name: order_svc
  user: order_app
  password: "${ORDER_DB_PASSWORD}"
  pool_size: 20
  max_overflow: 10
```

```yaml
# payment-service application config
database:
  host: mysql-payment.internal
  port: 3306
  name: payment_svc
  user: payment_app
  password: "${PAYMENT_DB_PASSWORD}"
  pool_size: 10
  max_overflow: 5
```

## Handling Queries That Span Services

When a view needs data from multiple services, use a dedicated read model or aggregation service that subscribes to events from each service and builds a consolidated read table:

```sql
-- order-query-service: read model aggregating data from events
USE order_query_svc;

CREATE TABLE order_summary (
  order_uuid    CHAR(36)      NOT NULL PRIMARY KEY,
  user_email    VARCHAR(200)  NOT NULL,
  user_name     VARCHAR(200)  NOT NULL,
  total_amount  DECIMAL(12,2) NOT NULL,
  payment_status VARCHAR(20),
  order_status  VARCHAR(20)   NOT NULL,
  created_at    TIMESTAMP     NOT NULL
) ENGINE=InnoDB;
```

## Summary

The database-per-service pattern with MySQL means each service gets its own MySQL database and a dedicated database user with access only to that database. Services communicate via APIs, never by cross-database queries. Use UUID natural keys for cross-service references, snapshot denormalized data in order lines and similar tables, and build dedicated read models for views that span multiple services. This pattern gives each service true schema and deployment independence.
