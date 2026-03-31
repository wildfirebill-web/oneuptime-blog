# How to Use FLOAT Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Float, Data Types, Floating Point, Numeric

Description: The MySQL FLOAT type stores single-precision floating-point numbers suitable for scientific data where approximate values and compact storage matter more than exact precision.

---

## What Is the FLOAT Data Type

`FLOAT` in MySQL is a single-precision floating-point number that uses 4 bytes of storage. It stores approximate numeric values with about 7 significant decimal digits of precision. Because it uses binary floating-point representation, `FLOAT` values are approximations rather than exact decimals.

Use `FLOAT` when:
- Approximate values are acceptable
- You are storing sensor readings, scientific measurements, or statistical values
- Storage space is a priority and you do not need the full precision of `DOUBLE`

Do not use `FLOAT` for money, financial calculations, or any data requiring exact precision. Use `DECIMAL` instead.

## FLOAT Syntax

```sql
FLOAT
FLOAT(p)
```

- `FLOAT` with no argument stores a 4-byte single-precision value
- `FLOAT(p)` where `p` is between 0-24 stores a 4-byte single-precision value
- `FLOAT(p)` where `p` is between 25-53 stores an 8-byte double-precision value (equivalent to DOUBLE)

Note: The precision parameter in `FLOAT(p)` specifies bits of mantissa precision, not decimal digits. In MySQL 8.0.17+, specifying precision with `FLOAT(M,D)` syntax is deprecated.

## Creating a Table with FLOAT Columns

```sql
CREATE TABLE sensor_readings (
  id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  sensor_id INT UNSIGNED NOT NULL,
  temperature FLOAT NOT NULL,
  humidity    FLOAT NOT NULL,
  pressure    FLOAT NOT NULL,
  recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Inserting and Querying FLOAT Values

```sql
INSERT INTO sensor_readings (sensor_id, temperature, humidity, pressure)
VALUES (1, 23.7, 65.3, 1013.25);

INSERT INTO sensor_readings (sensor_id, temperature, humidity, pressure)
VALUES (1, -5.2, 80.1, 998.50);

SELECT id, sensor_id, temperature, humidity, pressure
FROM sensor_readings;
```

Example output:

```text
+----+-----------+-------------+----------+----------+
| id | sensor_id | temperature | humidity | pressure |
+----+-----------+-------------+----------+----------+
|  1 |         1 |        23.7 |     65.3 |  1013.25 |
|  2 |         1 |        -5.2 |     80.1 |   998.50 |
+----+-----------+-------------+----------+----------+
```

## Understanding Floating-Point Imprecision

Because `FLOAT` uses binary representation, some decimal values cannot be stored exactly:

```sql
CREATE TABLE float_test (val FLOAT);
INSERT INTO float_test VALUES (0.1 + 0.2);
SELECT val FROM float_test;
```

```text
+--------------------+
| val                |
+--------------------+
| 0.30000001192092896|
+--------------------+
```

This is a fundamental property of binary floating-point, not a MySQL bug. Use `ROUND()` when displaying float values:

```sql
SELECT ROUND(temperature, 1) AS temp_rounded
FROM sensor_readings;
```

## FLOAT vs DOUBLE

| Property        | FLOAT          | DOUBLE         |
|-----------------|----------------|----------------|
| Storage         | 4 bytes        | 8 bytes        |
| Precision       | ~7 digits      | ~15-16 digits  |
| Range           | -3.4E38 to 3.4E38 | -1.7E308 to 1.7E308 |
| Use case        | Sensor data, low-precision measurements | Scientific computing, high-precision approximations |

When in doubt between FLOAT and DOUBLE, use DOUBLE. When in doubt between FLOAT/DOUBLE and exact values, use DECIMAL.

## Comparing FLOAT Values

Never use `=` to compare float values directly, as rounding errors make equality comparisons unreliable:

```sql
-- Unreliable - may not return the expected row
SELECT * FROM sensor_readings WHERE temperature = 23.7;

-- Reliable - use a range
SELECT * FROM sensor_readings
WHERE temperature BETWEEN 23.69 AND 23.71;

-- Or use ROUND for comparison
SELECT * FROM sensor_readings
WHERE ROUND(temperature, 1) = 23.7;
```

## Aggregating FLOAT Columns

Aggregate functions work normally with FLOAT, but the results are still approximate:

```sql
SELECT
  sensor_id,
  AVG(temperature)             AS avg_temp,
  ROUND(AVG(temperature), 2)   AS avg_temp_rounded,
  MIN(temperature)             AS min_temp,
  MAX(temperature)             AS max_temp
FROM sensor_readings
GROUP BY sensor_id;
```

## Storage Considerations

A `FLOAT` column uses 4 bytes regardless of the stored value. For tables with millions of rows containing sensor data, using `FLOAT` instead of `DOUBLE` saves 4 bytes per row per column - which can be significant at scale.

```sql
-- If precision beyond 7 digits is not needed, FLOAT saves space
CREATE TABLE telemetry (
  device_id INT UNSIGNED NOT NULL,
  lat  FLOAT NOT NULL,   -- GPS latitude: ~7 digit precision is sufficient
  lng  FLOAT NOT NULL,   -- GPS longitude: ~7 digit precision is sufficient
  alt  FLOAT,            -- altitude in meters
  ts   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX (device_id, ts)
);
```

## Summary

MySQL `FLOAT` stores single-precision floating-point numbers in 4 bytes, suitable for approximate numeric data like sensor readings and measurements. It provides about 7 significant digits of precision but is not appropriate for exact financial calculations - use `DECIMAL` for those. Always use range comparisons instead of equality checks when querying float columns, and apply `ROUND()` when displaying float values to users.
