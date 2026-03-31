# How to Use sqrt() and cbrt() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Math Function, Numeric, Distance Calculation

Description: Learn how sqrt() computes square roots and cbrt() computes cube roots in ClickHouse, with examples for Euclidean distance, geometric mean, and value normalization.

---

`sqrt()` and `cbrt()` are root functions in ClickHouse for computing square roots and cube roots respectively. `sqrt()` is one of the most commonly used math functions in analytical queries, appearing in distance calculations, standard deviation computations, and geometric algorithms. `cbrt()` is useful for geometric mean calculations, volume-to-side-length conversions, and any application where the cube root represents a meaningful transformation.

## Function Signatures

```text
sqrt(x)  -- returns the square root of x (x must be non-negative)
cbrt(x)  -- returns the cube root of x
```

Both functions return a Float64. `sqrt()` is undefined for negative inputs and will return NaN. `cbrt()` handles negative values correctly, returning a negative cube root.

## Basic Usage

```sql
SELECT
    sqrt(16)      AS sqrt_16,
    sqrt(2)       AS sqrt_2,
    cbrt(27)      AS cbrt_27,
    cbrt(8)       AS cbrt_8,
    cbrt(-27)     AS cbrt_neg;
```

Returns 4, 1.4142..., 3, 2, -3.

## Setting Up Sample Data for Distance Calculations

Create a table of 2D point coordinates to demonstrate Euclidean distance using `sqrt()`.

```sql
CREATE TABLE locations
(
    location_id  UInt64,
    location_name String,
    x            Float64,
    y            Float64
)
ENGINE = MergeTree
ORDER BY location_id;

INSERT INTO locations VALUES
(1, 'Warehouse A',  0.0,   0.0),
(2, 'Warehouse B',  10.0,  0.0),
(3, 'Customer X',   3.0,   4.0),
(4, 'Customer Y',   8.0,   6.0),
(5, 'Customer Z',   1.0,   7.0);
```

## Computing Euclidean Distance

Use `sqrt()` with the sum of squared differences to compute the straight-line distance between two points.

```sql
SELECT
    a.location_name AS from_loc,
    b.location_name AS to_loc,
    round(
        sqrt(pow(a.x - b.x, 2) + pow(a.y - b.y, 2)),
        2
    ) AS distance
FROM locations a
CROSS JOIN locations b
WHERE a.location_id < b.location_id
ORDER BY distance;
```

## Finding the Nearest Warehouse to Each Customer

Use `sqrt()` in a subquery to find the closest warehouse for each customer location.

```sql
SELECT
    c.location_name AS customer,
    argMin(w.location_name, sqrt(pow(c.x - w.x, 2) + pow(c.y - w.y, 2))) AS nearest_warehouse,
    round(min(sqrt(pow(c.x - w.x, 2) + pow(c.y - w.y, 2))), 2)           AS min_distance
FROM locations c
CROSS JOIN locations w
WHERE c.location_name LIKE 'Customer%'
  AND w.location_name LIKE 'Warehouse%'
GROUP BY customer;
```

## Computing Root Mean Square

The root mean square (RMS) is a statistical measure of magnitude. Use `sqrt()` combined with `avg()` to compute it.

```sql
CREATE TABLE sensor_readings
(
    sensor_id  UInt32,
    ts         DateTime,
    value      Float64
)
ENGINE = MergeTree
ORDER BY (sensor_id, ts);

INSERT INTO sensor_readings VALUES
(1, '2024-09-01 00:00:00', 2.5),
(1, '2024-09-01 00:01:00', -3.1),
(1, '2024-09-01 00:02:00', 1.8),
(1, '2024-09-01 00:03:00', -0.5),
(1, '2024-09-01 00:04:00', 4.2);
```

```sql
SELECT
    sensor_id,
    round(sqrt(avg(pow(value, 2))), 4) AS rms_value
FROM sensor_readings
GROUP BY sensor_id;
```

## Geometric Mean with cbrt()

The geometric mean of three values is the cube root of their product. Use `cbrt()` for three-value geometric means, or `exp(avg(log(x)))` for the general case.

```sql
SELECT
    round(cbrt(12.0 * 18.0 * 27.0), 4) AS geometric_mean_3;
```

## Normalizing by Square Root

Scale counts or totals by `sqrt()` to produce a size-normalized metric, commonly used in information retrieval (TF-IDF) and similarity scoring.

```sql
CREATE TABLE document_stats
(
    doc_id     UInt64,
    term       String,
    term_freq  UInt32,
    doc_length UInt32
)
ENGINE = MergeTree
ORDER BY (doc_id, term);

INSERT INTO document_stats VALUES
(1, 'database', 5, 120),
(1, 'query',    8, 120),
(2, 'database', 2, 40),
(2, 'index',    6, 40);
```

```sql
SELECT
    doc_id,
    term,
    term_freq,
    doc_length,
    round(term_freq / sqrt(doc_length), 4) AS normalized_freq
FROM document_stats;
```

## Summary

`sqrt()` is essential for distance calculations, RMS computations, standard deviation formulas, and geometric algorithms in ClickHouse. `cbrt()` provides cube roots for geometric means, volume calculations, and any domain where the cube root is a natural transformation. Both return Float64 and compose naturally with `pow()`, `avg()`, and other math functions to build complex expressions. When computing distances, consider `hypot()` as an alternative to the explicit `sqrt(pow(dx,2) + pow(dy,2))` pattern, as it handles overflow more robustly.
