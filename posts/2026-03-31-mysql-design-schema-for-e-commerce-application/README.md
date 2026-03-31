# How to Design a Schema for an E-Commerce Application in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, E-Commerce, Schema Design, Order Management

Description: Learn how to design a normalized e-commerce schema in MySQL covering products, inventory, orders, and payments with foreign keys.

---

An e-commerce schema must handle product catalogs, customer accounts, shopping carts, orders with line items, and payment records. Each entity has clear relationships and query patterns.

## Products and Inventory

```sql
CREATE TABLE products (
    id          INT UNSIGNED   NOT NULL AUTO_INCREMENT,
    sku         VARCHAR(100)   NOT NULL,
    name        VARCHAR(255)   NOT NULL,
    description TEXT,
    price       DECIMAL(10,2)  NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_sku (sku)
);

CREATE TABLE inventory (
    product_id   INT UNSIGNED NOT NULL,
    warehouse_id INT UNSIGNED NOT NULL DEFAULT 1,
    quantity     INT          NOT NULL DEFAULT 0,
    PRIMARY KEY (product_id, warehouse_id),
    CONSTRAINT fk_inv_product FOREIGN KEY (product_id) REFERENCES products (id)
);
```

## Customers

```sql
CREATE TABLE customers (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    email      VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name  VARCHAR(100) NOT NULL,
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_email (email)
);

CREATE TABLE addresses (
    id          INT UNSIGNED NOT NULL AUTO_INCREMENT,
    customer_id INT UNSIGNED NOT NULL,
    line1       VARCHAR(255) NOT NULL,
    city        VARCHAR(100) NOT NULL,
    country     CHAR(2)      NOT NULL,
    is_default  TINYINT(1)   NOT NULL DEFAULT 0,
    PRIMARY KEY (id),
    KEY idx_customer (customer_id),
    CONSTRAINT fk_addr_customer FOREIGN KEY (customer_id) REFERENCES customers (id) ON DELETE CASCADE
);
```

## Orders and Line Items

```sql
CREATE TABLE orders (
    id              INT UNSIGNED   NOT NULL AUTO_INCREMENT,
    customer_id     INT UNSIGNED   NOT NULL,
    shipping_addr_id INT UNSIGNED  NOT NULL,
    status          ENUM('pending','paid','shipped','delivered','cancelled') NOT NULL DEFAULT 'pending',
    total_amount    DECIMAL(12,2)  NOT NULL,
    placed_at       DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_customer_status (customer_id, status),
    KEY idx_placed_at (placed_at),
    CONSTRAINT fk_order_customer FOREIGN KEY (customer_id)     REFERENCES customers (id),
    CONSTRAINT fk_order_addr     FOREIGN KEY (shipping_addr_id) REFERENCES addresses (id)
);

CREATE TABLE order_items (
    id         INT UNSIGNED  NOT NULL AUTO_INCREMENT,
    order_id   INT UNSIGNED  NOT NULL,
    product_id INT UNSIGNED  NOT NULL,
    quantity   INT UNSIGNED  NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (id),
    KEY idx_order   (order_id),
    KEY idx_product (product_id),
    CONSTRAINT fk_oi_order   FOREIGN KEY (order_id)   REFERENCES orders   (id) ON DELETE CASCADE,
    CONSTRAINT fk_oi_product FOREIGN KEY (product_id) REFERENCES products (id)
);
```

## Payments

```sql
CREATE TABLE payments (
    id             INT UNSIGNED  NOT NULL AUTO_INCREMENT,
    order_id       INT UNSIGNED  NOT NULL,
    amount         DECIMAL(12,2) NOT NULL,
    method         ENUM('card','paypal','bank_transfer') NOT NULL,
    status         ENUM('pending','completed','failed','refunded') NOT NULL DEFAULT 'pending',
    paid_at        DATETIME      NULL,
    PRIMARY KEY (id),
    KEY idx_order (order_id),
    CONSTRAINT fk_pay_order FOREIGN KEY (order_id) REFERENCES orders (id)
);
```

## Common Queries

```sql
-- Recent orders with customer name and total
SELECT o.id, CONCAT(c.first_name, ' ', c.last_name) AS customer,
       o.total_amount, o.status, o.placed_at
FROM   orders o
JOIN   customers c ON c.id = o.customer_id
WHERE  o.placed_at >= CURDATE() - INTERVAL 7 DAY
ORDER BY o.placed_at DESC;

-- Top-selling products this month
SELECT p.name, SUM(oi.quantity) AS units_sold
FROM   order_items oi
JOIN   products p    ON p.id = oi.product_id
JOIN   orders   o    ON o.id = oi.order_id
WHERE  o.placed_at >= DATE_FORMAT(NOW(), '%Y-%m-01')
  AND  o.status != 'cancelled'
GROUP BY p.id
ORDER BY units_sold DESC
LIMIT 10;
```

## Summary

An e-commerce schema requires products with inventory, customers with addresses, orders with line items, and payments. Use `DECIMAL` for all monetary amounts. Index `(customer_id, status)` on orders for customer order history queries. Denormalize `unit_price` into order items so historical price accuracy is preserved even after product price changes.
