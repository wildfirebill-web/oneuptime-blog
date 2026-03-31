# How to Implement a State Machine in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Pattern, Schema, Trigger, Validation

Description: Learn how to implement a state machine in MySQL that enforces valid state transitions using constraints and triggers for workflow integrity.

---

## What Is a Database State Machine?

A state machine ensures an entity can only move between valid states in a defined order. Without enforcement, application bugs can corrupt state - orders can jump from "pending" to "shipped" without payment, or subscriptions can move from "cancelled" back to "active" incorrectly.

Enforcing transitions at the database level provides a safety net regardless of which application or service makes the change.

## Schema Design

```sql
-- Define all valid states and transitions
CREATE TABLE order_state_transitions (
    from_state VARCHAR(50) NOT NULL,
    to_state VARCHAR(50) NOT NULL,
    PRIMARY KEY (from_state, to_state)
);

-- Insert valid transitions for an order workflow
INSERT INTO order_state_transitions (from_state, to_state) VALUES
    ('pending', 'paid'),
    ('pending', 'cancelled'),
    ('paid', 'processing'),
    ('paid', 'refunded'),
    ('processing', 'shipped'),
    ('processing', 'refunded'),
    ('shipped', 'delivered'),
    ('shipped', 'returned'),
    ('delivered', 'returned'),
    ('cancelled', 'pending'),  -- Allow reactivation
    ('refunded', 'closed'),
    ('returned', 'closed');

-- Orders table with current state
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    total DECIMAL(10,2) NOT NULL,
    state VARCHAR(50) NOT NULL DEFAULT 'pending',
    state_changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_state (state),
    CONSTRAINT chk_state CHECK (state IN (
        'pending', 'paid', 'processing', 'shipped',
        'delivered', 'cancelled', 'refunded', 'returned', 'closed'
    ))
);
```

## Enforcing Transitions with a Trigger

```sql
DELIMITER $$

CREATE TRIGGER enforce_order_state_transition
BEFORE UPDATE ON orders
FOR EACH ROW
BEGIN
    -- Only validate if state is actually changing
    IF NEW.state != OLD.state THEN
        -- Check if the transition is valid
        IF NOT EXISTS (
            SELECT 1 FROM order_state_transitions
            WHERE from_state = OLD.state
              AND to_state = NEW.state
        ) THEN
            SIGNAL SQLSTATE '45000'
                SET MESSAGE_TEXT = CONCAT(
                    'Invalid state transition: ',
                    OLD.state, ' -> ', NEW.state
                );
        END IF;
    END IF;
END$$

DELIMITER ;
```

## State Transition with History

Track all transitions in an audit table:

```sql
CREATE TABLE order_state_history (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    from_state VARCHAR(50),
    to_state VARCHAR(50) NOT NULL,
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    notes TEXT,
    INDEX idx_order (order_id, changed_at),
    FOREIGN KEY (order_id) REFERENCES orders(id)
);

DELIMITER $$

CREATE TRIGGER log_order_state_change
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
    IF NEW.state != OLD.state THEN
        INSERT INTO order_state_history
            (order_id, from_state, to_state, changed_at)
        VALUES
            (NEW.id, OLD.state, NEW.state, NOW());
    END IF;
END$$

DELIMITER ;
```

## Transition Operations

```sql
-- Valid transition: pending -> paid
UPDATE orders SET state = 'paid' WHERE id = 1;

-- Invalid transition: triggers the enforcement trigger
UPDATE orders SET state = 'delivered' WHERE id = 1;
-- ERROR 1644 (45000): Invalid state transition: paid -> delivered

-- Query orders by state
SELECT id, customer_id, total, state, state_changed_at
FROM orders
WHERE state = 'processing'
ORDER BY state_changed_at;
```

## Stored Procedure for Transitions

```sql
DELIMITER $$

CREATE PROCEDURE transition_order(
    IN p_order_id INT,
    IN p_new_state VARCHAR(50),
    IN p_changed_by VARCHAR(100),
    IN p_notes TEXT
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLSTATE '45000'
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;

    UPDATE orders
    SET state = p_new_state
    WHERE id = p_order_id;

    -- Update the history with additional context
    UPDATE order_state_history
    SET changed_by = p_changed_by, notes = p_notes
    WHERE order_id = p_order_id
    ORDER BY changed_at DESC
    LIMIT 1;

    COMMIT;
END$$

DELIMITER ;

-- Usage
CALL transition_order(1, 'processing', 'payment-service', 'Payment confirmed');
```

## Summary

A MySQL state machine uses a transitions table as a whitelist of valid state changes, enforced via a `BEFORE UPDATE` trigger that signals an error on invalid transitions. Pairing this with an `AFTER UPDATE` trigger for history logging gives you complete auditability of every state change. This database-level enforcement ensures workflow integrity regardless of which application code attempts the state change.
