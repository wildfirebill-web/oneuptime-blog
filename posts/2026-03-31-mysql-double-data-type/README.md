# How to Use DOUBLE Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mysql, Double, Floating Point, Data Types, Numeric

Description: The MySQL DOUBLE data type stores double-precision floating-point numbers with approximately 15-16 significant digits, suitable for scientific and engineering calculations.

---

## What Is the DOUBLE Data Type

In MySQL, `DOUBLE` (also written as `DOUBLE PRECISION` or `REAL`) is a floating-point data type that stores double-precision numbers conforming to the IEEE 754 standard. It uses 8 bytes of storage and supports values ranging from approximately `-1.7976931348623157E+308` to `1.7976931348623157E+308` for non-zero values, with precision of about 15 to 16 significant decimal digits.

`DOUBLE` is suitable for scientific, statistical, and engineering data where you need a wide range of values and approximate precision is acceptable.

## DOUBLE vs FLOAT vs DECIMAL

| Type | Storage | Precision | Use Case |
|---|---|---|---|
| `FLOAT` | 4 bytes | ~7 significant digits | Low precision, large range |
| `DOUBLE` | 8 bytes | ~15-16 significant digits | High precision, large range |
| `DECIMAL(p,s)` | Variable | Exact | Money, exact values |

The key difference from `DECIMAL` is that `DOUBLE` uses binary floating-point representation, which means it cannot represent all decimal fractions exactly. For example, `0.1` cannot be stored precisely in binary floating-point. Use `DECIMAL` when exactness matters (financial data).

## Creating a Table with DOUBLE

```sql
CREATE TABLE measurements (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sensor_id INT NOT NULL,
    temperature DOUBLE,
    pressure DOUBLE,
    voltage DOUBLE NOT NULL DEFAULT 0.0,
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Inserting DOUBLE Values

```sql
INSERT INTO measurements (sensor_id, temperature, pressure, voltage)
VALUES
    (1, 23.456789012345, 101325.0, 3.14159265358979),
    (2, -273.15, 0.0, 0.000001),
    (3, 6.674e-11, 1.38064852e-23, 2.99792458e8);
```

## Querying DOUBLE Values

```sql
SELECT
    sensor_id,
    temperature,
    pressure,
    FORMAT(voltage, 10) AS voltage_formatted
FROM measurements;
```

## Arithmetic with DOUBLE

```sql
-- Calculate kinetic energy: 0.5 * m * v^2
SELECT
    sensor_id,
    0.5 * 10.0 * POW(voltage, 2) AS kinetic_energy,
    SQRT(pressure) AS sqrt_pressure
FROM measurements;
```

## Floating-Point Precision Caveats

Because `DOUBLE` uses binary floating-point, be cautious with exact equality comparisons:

```sql
-- This may return unexpected results due to floating-point representation
SELECT * FROM measurements WHERE temperature = 23.456789012345;

-- Better: use a range comparison
SELECT * FROM measurements
WHERE temperature BETWEEN 23.4567890 AND 23.4567891;
```

A classic demonstration of floating-point imprecision:

```sql
SELECT 0.1 + 0.2;
-- Returns: 0.30000000000000004
```

## DOUBLE with Display Width (Deprecated)

In older MySQL documentation you may see `DOUBLE(M, D)` syntax where `M` is the total display width and `D` is the number of decimal places. This syntax is deprecated in MySQL 8.0.17 and should not be used in new code:

```sql
-- Deprecated syntax, avoid in new code
CREATE TABLE old_style (val DOUBLE(10, 4));
```

Instead, use plain `DOUBLE` and handle display formatting in your application or with the `FORMAT()` function.

## Using DOUBLE with Aggregate Functions

```sql
-- Statistical aggregation on DOUBLE columns
SELECT
    sensor_id,
    COUNT(*) AS readings,
    AVG(temperature) AS avg_temp,
    MIN(temperature) AS min_temp,
    MAX(temperature) AS max_temp,
    STDDEV(temperature) AS stddev_temp
FROM measurements
GROUP BY sensor_id;
```

## Storing Scientific Values

`DOUBLE` handles scientific notation well, making it useful for physics, chemistry, and astronomy data:

```sql
CREATE TABLE constants (
    name VARCHAR(100) NOT NULL,
    symbol VARCHAR(20),
    value DOUBLE NOT NULL,
    unit VARCHAR(50)
);

INSERT INTO constants (name, symbol, value, unit)
VALUES
    ('Speed of light', 'c', 2.99792458e8, 'm/s'),
    ('Gravitational constant', 'G', 6.67430e-11, 'm3 kg-1 s-2'),
    ('Boltzmann constant', 'k', 1.380649e-23, 'J/K'),
    ('Planck constant', 'h', 6.62607015e-34, 'J*s');

SELECT name, symbol, value FROM constants;
```

## DOUBLE vs REAL vs DOUBLE PRECISION

In MySQL, `REAL` and `DOUBLE PRECISION` are synonyms for `DOUBLE` unless `REAL_AS_FLOAT` SQL mode is enabled:

```sql
-- All three are equivalent (without REAL_AS_FLOAT mode)
CREATE TABLE example (
    a DOUBLE,
    b REAL,
    c DOUBLE PRECISION
);
```

## Summary

`DOUBLE` in MySQL is an 8-byte double-precision floating-point type providing approximately 15-16 significant digits of precision, suitable for scientific calculations, sensor data, and large numeric ranges. Unlike `DECIMAL`, it uses binary floating-point representation and cannot store all decimal fractions exactly, so you should avoid exact equality comparisons and use range queries instead. For financial or monetary calculations where exactness is required, use `DECIMAL` instead.
