# How to Implement Slowly Changing Dimensions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Slowly Changing Dimension, Data Warehouse, ETL

Description: Learn how to implement Type 1, Type 2, and Type 3 slowly changing dimensions in MySQL to track historical changes in dimension attributes over time.

---

## What Are Slowly Changing Dimensions

Slowly Changing Dimensions (SCDs) describe how a data warehouse handles changes to dimension attributes over time. A customer moving cities, a product being renamed, or an employee changing departments are all examples of slowly changing data. MySQL supports all common SCD types through standard SQL.

## Type 1 - Overwrite

Type 1 simply overwrites the old value. No history is kept. Use this when history is irrelevant.

```sql
CREATE TABLE dim_customer (
  customer_key BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id  VARCHAR(50)  NOT NULL,
  full_name    VARCHAR(200) NOT NULL,
  city         VARCHAR(100) NOT NULL,
  country      VARCHAR(100) NOT NULL,
  UNIQUE KEY uq_customer_id (customer_id)
) ENGINE=InnoDB;

-- Type 1 update: overwrite city
UPDATE dim_customer
SET city = 'Denver'
WHERE customer_id = 'C-1001';
```

## Type 2 - Add New Row

Type 2 preserves history by inserting a new row when a dimension attribute changes. Use a surrogate key, a natural key, effective date columns, and a current flag.

```sql
CREATE TABLE dim_customer_scd2 (
  customer_key  BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id   VARCHAR(50)  NOT NULL,
  full_name     VARCHAR(200) NOT NULL,
  city          VARCHAR(100) NOT NULL,
  country       VARCHAR(100) NOT NULL,
  effective_from DATE        NOT NULL,
  effective_to   DATE,                  -- NULL means current
  is_current     TINYINT(1)  NOT NULL DEFAULT 1,
  INDEX idx_customer_id  (customer_id),
  INDEX idx_current      (customer_id, is_current)
) ENGINE=InnoDB;

-- Step 1: expire the old row
UPDATE dim_customer_scd2
SET is_current   = 0,
    effective_to = CURDATE() - INTERVAL 1 DAY
WHERE customer_id = 'C-1001'
  AND is_current  = 1;

-- Step 2: insert the new row
INSERT INTO dim_customer_scd2
  (customer_id, full_name, city, country, effective_from, effective_to, is_current)
VALUES
  ('C-1001', 'Alice Brown', 'Denver', 'US', CURDATE(), NULL, 1);
```

Query current records only:

```sql
SELECT *
FROM dim_customer_scd2
WHERE is_current = 1;
```

Query the historical record at a specific date:

```sql
SELECT *
FROM dim_customer_scd2
WHERE customer_id    = 'C-1001'
  AND effective_from <= '2024-06-15'
  AND (effective_to  >= '2024-06-15' OR effective_to IS NULL);
```

## Type 3 - Add New Column

Type 3 stores the current and previous value in separate columns. This tracks one level of history per attribute.

```sql
CREATE TABLE dim_customer_scd3 (
  customer_key   BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id    VARCHAR(50)  NOT NULL,
  full_name      VARCHAR(200) NOT NULL,
  current_city   VARCHAR(100) NOT NULL,
  previous_city  VARCHAR(100),
  city_changed_at DATE,
  UNIQUE KEY uq_customer_id (customer_id)
) ENGINE=InnoDB;

-- Update with Type 3 logic
UPDATE dim_customer_scd3
SET previous_city  = current_city,
    current_city   = 'Portland',
    city_changed_at = CURDATE()
WHERE customer_id = 'C-1001';
```

## Choosing the Right Type

```text
Type 1 - history not needed, simple overwrite
Type 2 - full history needed for point-in-time reporting
Type 3 - only previous value matters, limited history acceptable
```

## Automating SCD2 with a Stored Procedure

```sql
DELIMITER $$
CREATE PROCEDURE upsert_customer_scd2(
  IN p_customer_id VARCHAR(50),
  IN p_full_name   VARCHAR(200),
  IN p_city        VARCHAR(100),
  IN p_country     VARCHAR(100)
)
BEGIN
  DECLARE v_exists INT DEFAULT 0;

  SELECT COUNT(*) INTO v_exists
  FROM dim_customer_scd2
  WHERE customer_id = p_customer_id AND is_current = 1
    AND city = p_city AND country = p_country;

  IF v_exists = 0 THEN
    UPDATE dim_customer_scd2
    SET is_current = 0, effective_to = CURDATE() - INTERVAL 1 DAY
    WHERE customer_id = p_customer_id AND is_current = 1;

    INSERT INTO dim_customer_scd2
      (customer_id, full_name, city, country, effective_from, is_current)
    VALUES
      (p_customer_id, p_full_name, p_city, p_country, CURDATE(), 1);
  END IF;
END$$
DELIMITER ;
```

## Summary

MySQL supports all SCD types through standard SQL. Type 1 is the simplest; Type 2 provides full history with effective date ranges and a current flag; Type 3 tracks only one previous value. Automate SCD2 logic with a stored procedure or ETL script to keep the dimension consistent as source data changes.
