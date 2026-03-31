# How to Implement CQRS with MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CQRS, Architecture, Query, Command

Description: Learn how to implement the CQRS pattern with MySQL by separating read and write models using dedicated tables, views, and replication.

---

## What Is CQRS?

Command Query Responsibility Segregation (CQRS) separates read operations (queries) from write operations (commands). In MySQL, this means maintaining a write-optimized model for commands and a read-optimized model for queries. This pattern improves scalability and allows each side to be tuned independently.

## Setting Up the Write Model

The write model holds normalized, consistent data. Design it with strict constraints and proper indexes for writes:

```sql
CREATE TABLE orders (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  status ENUM('pending','confirmed','shipped','cancelled') NOT NULL DEFAULT 'pending',
  total_amount DECIMAL(10,2) NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_customer (customer_id),
  INDEX idx_status (status)
) ENGINE=InnoDB;

CREATE TABLE order_items (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  order_id BIGINT UNSIGNED NOT NULL,
  product_id BIGINT UNSIGNED NOT NULL,
  quantity INT UNSIGNED NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL,
  FOREIGN KEY (order_id) REFERENCES orders(id)
) ENGINE=InnoDB;
```

## Building the Read Model

The read model is denormalized for fast queries. Use a summary table that gets updated via triggers or application logic:

```sql
CREATE TABLE order_summaries (
  order_id BIGINT UNSIGNED PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  customer_name VARCHAR(200),
  status VARCHAR(50) NOT NULL,
  item_count INT UNSIGNED NOT NULL DEFAULT 0,
  total_amount DECIMAL(10,2) NOT NULL,
  created_at DATETIME NOT NULL,
  INDEX idx_customer_status (customer_id, status),
  INDEX idx_created (created_at)
) ENGINE=InnoDB;
```

## Synchronizing Write to Read Model with Triggers

Use MySQL triggers to keep the read model in sync after writes:

```sql
DELIMITER $$

CREATE TRIGGER after_order_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
  INSERT INTO order_summaries (order_id, customer_id, status, item_count, total_amount, created_at)
  VALUES (NEW.id, NEW.customer_id, NEW.status, 0, NEW.total_amount, NEW.created_at);
END$$

CREATE TRIGGER after_order_update
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
  UPDATE order_summaries
  SET status = NEW.status,
      total_amount = NEW.total_amount
  WHERE order_id = NEW.id;
END$$

DELIMITER ;
```

## Using MySQL Replication for CQRS

For larger systems, configure a read replica to serve query traffic. Point your write model at the primary and your read model at the replica:

```sql
-- On the primary: issue commands
INSERT INTO orders (customer_id, total_amount) VALUES (101, 299.99);

-- On the replica: issue queries
SELECT * FROM order_summaries WHERE customer_id = 101 ORDER BY created_at DESC LIMIT 10;
```

## Application-Layer CQRS

At the application layer, enforce the separation with dedicated database connections:

```python
import mysql.connector

write_db = mysql.connector.connect(host="primary-db", user="app", password="secret", database="shop")
read_db  = mysql.connector.connect(host="replica-db",  user="app", password="secret", database="shop")

def create_order(customer_id, total):
    cur = write_db.cursor()
    cur.execute("INSERT INTO orders (customer_id, total_amount) VALUES (%s, %s)", (customer_id, total))
    write_db.commit()

def get_customer_orders(customer_id):
    cur = read_db.cursor(dictionary=True)
    cur.execute("SELECT * FROM order_summaries WHERE customer_id = %s", (customer_id,))
    return cur.fetchall()
```

## Handling Eventual Consistency

When using replication, reads may lag behind writes. Mitigate this by reading from the primary for time-sensitive queries immediately after a write, or by tracking replication lag:

```sql
SHOW REPLICA STATUS\G
-- Check Seconds_Behind_Source value
```

## Summary

CQRS in MySQL separates write operations (on normalized tables) from read operations (on denormalized summary tables or replicas). Use triggers for lightweight synchronization, MySQL replication for scalable read scaling, and dedicated application connections to enforce the boundary. This pattern reduces query contention and allows both sides to be independently optimized.
