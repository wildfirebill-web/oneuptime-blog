# How to Create Snowflake Schema in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Snowflake Schema, Data Warehouse, Normalization

Description: Learn how to implement a snowflake schema in MySQL by normalizing dimension tables into sub-dimensions, reducing redundancy while supporting analytical queries.

---

## What Is a Snowflake Schema

A snowflake schema extends the star schema by normalizing dimension tables into multiple related sub-dimension tables. For example, a `dim_product` that stores category information might be split into `dim_product` and `dim_category`. This reduces data redundancy and storage but requires additional joins in queries.

## Star vs Snowflake

| Aspect | Star Schema | Snowflake Schema |
|---|---|---|
| Normalization | Denormalized dimensions | Normalized sub-dimensions |
| Storage | More redundancy | Less redundancy |
| Query joins | Fewer | More |
| ETL complexity | Simpler | More complex |

Use snowflake schema when dimension tables are large and frequently updated, or when storage cost matters.

## Building the Schema

```sql
-- Category sub-dimension
CREATE TABLE dim_category (
  category_key  INT          NOT NULL AUTO_INCREMENT PRIMARY KEY,
  category_name VARCHAR(100) NOT NULL,
  department    VARCHAR(100) NOT NULL,
  UNIQUE KEY uq_category (category_name, department)
) ENGINE=InnoDB;

-- Product dimension references dim_category
CREATE TABLE dim_product (
  product_key  BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  sku          VARCHAR(50)   NOT NULL,
  product_name VARCHAR(200)  NOT NULL,
  category_key INT           NOT NULL,
  unit_cost    DECIMAL(10,2) NOT NULL,
  UNIQUE KEY uq_sku (sku),
  INDEX idx_category (category_key),
  CONSTRAINT fk_prod_cat FOREIGN KEY (category_key)
    REFERENCES dim_category(category_key)
) ENGINE=InnoDB;

-- Geography sub-dimension
CREATE TABLE dim_geography (
  geo_key    INT          NOT NULL AUTO_INCREMENT PRIMARY KEY,
  city       VARCHAR(100) NOT NULL,
  state      VARCHAR(100) NOT NULL,
  country    VARCHAR(100) NOT NULL,
  region     VARCHAR(50)  NOT NULL,
  UNIQUE KEY uq_geo (city, state, country)
) ENGINE=InnoDB;

-- Customer dimension references dim_geography
CREATE TABLE dim_customer (
  customer_key BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id  VARCHAR(50)  NOT NULL,
  full_name    VARCHAR(200) NOT NULL,
  geo_key      INT          NOT NULL,
  segment      VARCHAR(50)  NOT NULL,
  UNIQUE KEY uq_customer_id (customer_id),
  INDEX idx_geo (geo_key),
  CONSTRAINT fk_cust_geo FOREIGN KEY (geo_key)
    REFERENCES dim_geography(geo_key)
) ENGINE=InnoDB;

-- Fact table
CREATE TABLE fact_sales (
  sale_id      BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  date_key     INT           NOT NULL,
  product_key  BIGINT        NOT NULL,
  customer_key BIGINT        NOT NULL,
  quantity     INT           NOT NULL,
  revenue      DECIMAL(12,2) NOT NULL,
  INDEX idx_date     (date_key),
  INDEX idx_product  (product_key),
  INDEX idx_customer (customer_key)
) ENGINE=InnoDB;
```

## Loading Sub-Dimensions First

Always insert parent tables before child tables to satisfy foreign keys:

```sql
-- 1. Insert geography
INSERT INTO dim_geography (city, state, country, region)
VALUES ('Seattle', 'WA', 'US', 'North America')
ON DUPLICATE KEY UPDATE region = VALUES(region);

-- 2. Resolve geography key, then insert customer
INSERT INTO dim_customer (customer_id, full_name, geo_key, segment)
SELECT 'C-2001', 'Bob Lee', geo_key, 'SMB'
FROM dim_geography
WHERE city = 'Seattle' AND state = 'WA' AND country = 'US'
ON DUPLICATE KEY UPDATE segment = VALUES(segment);
```

## Querying the Snowflake Schema

Queries need extra joins to traverse the normalization hierarchy:

```sql
SELECT
  g.region,
  c.category_name,
  SUM(f.revenue)  AS total_revenue,
  COUNT(*)        AS order_count
FROM fact_sales     f
JOIN dim_customer   cu ON cu.customer_key = f.customer_key
JOIN dim_geography  g  ON g.geo_key       = cu.geo_key
JOIN dim_product    p  ON p.product_key   = f.product_key
JOIN dim_category   c  ON c.category_key  = p.category_key
GROUP BY g.region, c.category_name
ORDER BY total_revenue DESC;
```

## Performance Considerations

Each extra join adds overhead. To keep queries fast:
- Index all foreign key columns in dimension tables
- Keep sub-dimension tables small enough to fit in buffer pool
- Use `STRAIGHT_JOIN` if MySQL chooses a suboptimal join order

```sql
-- Force join order: fact -> product -> category -> customer -> geo
SELECT STRAIGHT_JOIN g.region, c.category_name, SUM(f.revenue)
FROM fact_sales f
JOIN dim_product p  ON p.product_key  = f.product_key
JOIN dim_category c ON c.category_key = p.category_key
JOIN dim_customer cu ON cu.customer_key = f.customer_key
JOIN dim_geography g ON g.geo_key       = cu.geo_key
GROUP BY g.region, c.category_name;
```

## Summary

Snowflake schema normalizes dimension tables to reduce redundancy at the cost of more joins. In MySQL, this pattern works well when dimensions have large repeated attribute sets. Keep all foreign keys indexed, load parent tables before child tables during ETL, and consider query hints when the optimizer misjudges join order on complex hierarchies.
