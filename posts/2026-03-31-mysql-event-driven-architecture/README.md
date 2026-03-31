# How to Implement Event-Driven Architecture with MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Event-Driven, Outbox Pattern, Trigger, Architecture

Description: Learn how to implement event-driven architecture with MySQL using the outbox pattern, triggers, and polling relays to produce reliable domain events from database writes.

---

## What Is Event-Driven Architecture?

In an event-driven architecture, services communicate by producing and consuming events rather than making direct calls. MySQL becomes an event source when database changes - order created, payment received, user registered - are published as events to other systems.

## The Outbox Pattern for Reliable Events

The core challenge is atomicity: a service must persist data AND publish an event without risk of one succeeding while the other fails. The outbox pattern solves this by writing both in the same transaction:

```sql
CREATE TABLE domain_events (
  id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  aggregate_type VARCHAR(100) NOT NULL,
  aggregate_id   BIGINT UNSIGNED NOT NULL,
  event_type     VARCHAR(200) NOT NULL,
  payload        JSON NOT NULL,
  status         ENUM('pending','dispatched') NOT NULL DEFAULT 'pending',
  created_at     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  dispatched_at  DATETIME DEFAULT NULL,
  INDEX idx_status_created (status, created_at)
) ENGINE=InnoDB;
```

Application code writes both within a transaction:

```sql
START TRANSACTION;

INSERT INTO orders (customer_id, total_amount, status)
VALUES (42, 199.99, 'confirmed');

SET @order_id = LAST_INSERT_ID();

INSERT INTO domain_events (aggregate_type, aggregate_id, event_type, payload)
VALUES (
  'Order',
  @order_id,
  'OrderConfirmed',
  JSON_OBJECT('order_id', @order_id, 'customer_id', 42, 'amount', 199.99)
);

COMMIT;
```

## Trigger-Based Event Capture

For legacy systems where you cannot change application code, use MySQL triggers:

```sql
DELIMITER $$
CREATE TRIGGER after_order_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
  INSERT INTO domain_events (aggregate_type, aggregate_id, event_type, payload)
  VALUES (
    'Order',
    NEW.id,
    'OrderCreated',
    JSON_OBJECT('order_id', NEW.id, 'customer_id', NEW.customer_id,
                'amount', NEW.total_amount, 'status', NEW.status)
  );
END$$

CREATE TRIGGER after_order_update
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
  IF OLD.status != NEW.status THEN
    INSERT INTO domain_events (aggregate_type, aggregate_id, event_type, payload)
    VALUES (
      'Order',
      NEW.id,
      CONCAT('OrderStatus', NEW.status),
      JSON_OBJECT('order_id', NEW.id, 'old_status', OLD.status, 'new_status', NEW.status)
    );
  END IF;
END$$
DELIMITER ;
```

## Event Relay to a Message Broker

A relay process reads pending events and dispatches them:

```python
import mysql.connector, json, pika

conn = mysql.connector.connect(host="db", user="app", password="secret", database="myapp")
rmq_channel = setup_rabbitmq()

cur = conn.cursor(dictionary=True)
conn.start_transaction()

cur.execute("""
  SELECT id, event_type, aggregate_id, payload
  FROM domain_events
  WHERE status = 'pending'
  ORDER BY id ASC LIMIT 50
  FOR UPDATE SKIP LOCKED
""")

for event in cur.fetchall():
    rmq_channel.basic_publish(
        exchange='domain-events',
        routing_key=event['event_type'],
        body=json.dumps(event['payload']),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    cur.execute(
        "UPDATE domain_events SET status='dispatched', dispatched_at=NOW() WHERE id=%s",
        (event['id'],)
    )

conn.commit()
```

## Event Consumer Example

```sql
CREATE TABLE order_notifications (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  order_id   BIGINT UNSIGNED NOT NULL,
  message    TEXT NOT NULL,
  sent_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

Consumers insert into projection tables when they receive events, keeping their own read models up to date.

## Summary

Implement event-driven architecture with MySQL using the outbox pattern - writing events to a `domain_events` table in the same transaction as business data. Use triggers for legacy code paths. A relay process polls pending events with `SELECT FOR UPDATE SKIP LOCKED` and publishes to a message broker. This guarantees at-least-once delivery without distributed transactions.
