# How to Choose Between Stored Procedures and Application Logic in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Architecture, Best Practice, Performance

Description: Learn when to use MySQL stored procedures versus application-layer logic by comparing maintainability, performance, testability, and deployment trade-offs.

---

Stored procedures move logic into the database while application-layer code keeps it in your programming language of choice. Both approaches have legitimate use cases, but the trade-offs around deployability, testability, and maintainability usually favor application code for business logic.

## What Stored Procedures Do Well

Stored procedures excel for operations that are tightly coupled to the database and benefit from executing close to the data:

```sql
DELIMITER $$

CREATE PROCEDURE close_expired_sessions()
BEGIN
  -- Bulk update with no network round-trips per row
  UPDATE user_sessions
  SET status = 'expired', expired_at = NOW()
  WHERE status = 'active'
    AND last_activity_at < NOW() - INTERVAL 30 MINUTE;

  -- Log the count of affected rows
  INSERT INTO maintenance_log (operation, affected_rows, run_at)
  VALUES ('close_expired_sessions', ROW_COUNT(), NOW());
END $$

DELIMITER ;
```

Scheduled maintenance jobs like this benefit from running inside the database because they avoid fetching rows to the application layer only to send them back as updates.

## What Application Code Does Better

Business logic that involves external systems, complex conditional branching, or frequent changes belongs in application code:

```python
# Business rule: apply discount if customer is premium and order exceeds threshold
def calculate_order_total(customer_id, line_items):
    customer = customer_service.get(customer_id)
    subtotal = sum(item.price * item.quantity for item in line_items)

    discount = 0
    if customer.tier == 'premium' and subtotal > 200:
        discount = subtotal * 0.10

    tax = tax_service.calculate(subtotal - discount, customer.country)
    return subtotal - discount + tax
```

Testing this function in isolation with mocked dependencies is straightforward. Testing the equivalent stored procedure requires a real MySQL instance and test data setup.

## Deployment and Version Control

Stored procedures live in the database, separate from your application repository. Changes require coordinated deployments:

```sql
-- Migration file: V15__update_close_expired_sessions.sql
DROP PROCEDURE IF EXISTS close_expired_sessions;

DELIMITER $$
CREATE PROCEDURE close_expired_sessions()
BEGIN
  -- Updated logic here
END $$
DELIMITER ;
```

Track stored procedures in migration files alongside schema changes. Tools like Flyway and Liquibase support this. Never modify stored procedures directly in production without a corresponding migration file.

## When Each Approach Is Appropriate

Use stored procedures for:
- Bulk data maintenance jobs that would require fetching large result sets to application memory
- Enforcing complex cross-table constraints beyond what CHECK constraints support
- Encapsulating legacy logic during an incremental migration

Use application code for:
- Business rules that depend on external services or configuration
- Logic that requires unit tests with mocked dependencies
- Any rule that changes frequently with product requirements

## Calling Stored Procedures from Application Code

When you do use stored procedures, call them cleanly:

```python
with conn.cursor() as cur:
    cur.callproc('close_expired_sessions')
    conn.commit()
```

```javascript
await pool.execute('CALL close_expired_sessions()');
```

## Summary

Stored procedures are a good fit for database-internal bulk operations and scheduled maintenance. Application code is better for business logic that changes often, requires external dependencies, or needs to be unit tested. The key decision factor is where the knowledge lives: if the logic only makes sense in the context of database rows and tables, a stored procedure is reasonable; if it depends on application context or external systems, keep it in application code.
