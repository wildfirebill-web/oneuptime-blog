# How to Design a Schema for an Inventory System in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Inventory, Schema Design, Stock Management

Description: Learn how to design a complete inventory management schema in MySQL covering products, warehouses, stock movements, and low-stock alerts.

---

An inventory system tracks what products exist, where they are stored, how stock levels change, and when to reorder. The schema must support accurate stock counts and movement history.

## Products and Categories

```sql
CREATE TABLE categories (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE products (
    id          INT UNSIGNED  NOT NULL AUTO_INCREMENT,
    category_id INT UNSIGNED  NULL,
    sku         VARCHAR(100)  NOT NULL,
    name        VARCHAR(255)  NOT NULL,
    description TEXT          NULL,
    unit_cost   DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    reorder_qty INT UNSIGNED  NOT NULL DEFAULT 0,
    PRIMARY KEY (id),
    UNIQUE KEY uq_sku (sku),
    KEY idx_category (category_id),
    CONSTRAINT fk_prod_cat FOREIGN KEY (category_id) REFERENCES categories (id) ON DELETE SET NULL
);
```

## Warehouses and Locations

```sql
CREATE TABLE warehouses (
    id      INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name    VARCHAR(150) NOT NULL,
    address VARCHAR(255) NULL,
    PRIMARY KEY (id)
);

CREATE TABLE storage_locations (
    id           INT UNSIGNED NOT NULL AUTO_INCREMENT,
    warehouse_id INT UNSIGNED NOT NULL,
    aisle        VARCHAR(20)  NOT NULL,
    shelf        VARCHAR(20)  NOT NULL,
    PRIMARY KEY (id),
    KEY idx_warehouse (warehouse_id),
    CONSTRAINT fk_loc_wh FOREIGN KEY (warehouse_id) REFERENCES warehouses (id) ON DELETE CASCADE
);
```

## Stock Levels

```sql
CREATE TABLE stock_levels (
    product_id    INT UNSIGNED NOT NULL,
    location_id   INT UNSIGNED NOT NULL,
    quantity      INT          NOT NULL DEFAULT 0,
    last_updated  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (product_id, location_id),
    CONSTRAINT fk_sl_product  FOREIGN KEY (product_id)  REFERENCES products          (id),
    CONSTRAINT fk_sl_location FOREIGN KEY (location_id) REFERENCES storage_locations (id)
);
```

## Stock Movements (Ledger)

```sql
CREATE TABLE stock_movements (
    id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    product_id    INT UNSIGNED    NOT NULL,
    location_id   INT UNSIGNED    NOT NULL,
    movement_type ENUM('receipt','shipment','adjustment','transfer') NOT NULL,
    quantity      INT             NOT NULL,
    reference     VARCHAR(100)    NULL,
    moved_by      INT UNSIGNED    NULL,
    moved_at      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_product_moved (product_id, moved_at),
    KEY idx_location      (location_id),
    CONSTRAINT fk_sm_product  FOREIGN KEY (product_id)  REFERENCES products          (id),
    CONSTRAINT fk_sm_location FOREIGN KEY (location_id) REFERENCES storage_locations (id)
);
```

## Updating Stock With a Trigger

```sql
DELIMITER $$
CREATE TRIGGER trg_update_stock AFTER INSERT ON stock_movements
FOR EACH ROW
BEGIN
    INSERT INTO stock_levels (product_id, location_id, quantity)
    VALUES (NEW.product_id, NEW.location_id, NEW.quantity)
    ON DUPLICATE KEY UPDATE quantity = quantity + NEW.quantity;
END$$
DELIMITER ;
```

## Querying Stock and Low-Stock Alerts

```sql
-- Current stock per product across all locations
SELECT p.sku, p.name, SUM(sl.quantity) AS total_stock
FROM   products p
JOIN   stock_levels sl ON sl.product_id = p.id
GROUP BY p.id
ORDER BY total_stock;

-- Products below reorder threshold
SELECT p.sku, p.name, SUM(sl.quantity) AS stock, p.reorder_qty
FROM   products p
JOIN   stock_levels sl ON sl.product_id = p.id
GROUP BY p.id
HAVING stock < p.reorder_qty;
```

## Summary

An inventory schema separates product definitions from stock levels and movement history. The `stock_movements` table acts as an append-only ledger. A trigger (or application code) updates the `stock_levels` summary table on each movement. Query the ledger for auditing and the summary table for current stock counts and reorder alerts.
