# How to Use SIN(), COS(), TAN() Trigonometric Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Trigonometry, Sin, Cos, Tan, Math Functions, Sql

Description: Learn how to use MySQL's SIN(), COS(), and TAN() functions for trigonometric calculations, including practical geospatial distance examples.

---

## Overview

MySQL provides a full set of trigonometric functions including `SIN()`, `COS()`, and `TAN()`. These functions accept an angle expressed in radians and return the corresponding trigonometric ratio. They are useful for geometric calculations, signal processing queries, and distance calculations.

## Basic Syntax

```sql
SIN(X)   -- sine of X radians
COS(X)   -- cosine of X radians
TAN(X)   -- tangent of X radians
```

All three functions take a single `DOUBLE` argument in radians and return a `DOUBLE`.

## Converting Degrees to Radians

MySQL does not accept degrees directly. Use `PI()` to convert:

```sql
-- Degrees to radians: degrees * PI() / 180
SELECT RADIANS(90);             -- 1.5707963267949
SELECT 90 * PI() / 180;        -- equivalent

-- Radians to degrees
SELECT DEGREES(PI() / 2);      -- 90
```

## Basic Trigonometric Examples

```sql
-- SIN
SELECT SIN(0);            -- 0
SELECT SIN(PI() / 2);     -- 1 (sin of 90 degrees)
SELECT SIN(PI());         -- ~0 (floating point rounding)

-- COS
SELECT COS(0);            -- 1
SELECT COS(PI() / 2);     -- ~0 (cos of 90 degrees)
SELECT COS(PI());         -- -1

-- TAN
SELECT TAN(0);            -- 0
SELECT TAN(PI() / 4);     -- 1 (tan of 45 degrees)
```

## Computing Distance with the Haversine Formula

```sql
-- Calculate great-circle distance between two GPS coordinates
-- Using the Haversine formula
SELECT
  6371 * 2 * ASIN(
    SQRT(
      POW(SIN(RADIANS(lat2 - lat1) / 2), 2) +
      COS(RADIANS(lat1)) * COS(RADIANS(lat2)) *
      POW(SIN(RADIANS(lon2 - lon1) / 2), 2)
    )
  ) AS distance_km
FROM (
  SELECT 40.7128 AS lat1, -74.0060 AS lon1,   -- New York
         51.5074 AS lat2, -0.1278  AS lon2    -- London
) coords;
```

## Storing Coordinates and Querying by Distance

```sql
CREATE TABLE locations (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  lat DOUBLE,
  lon DOUBLE
);

INSERT INTO locations (name, lat, lon) VALUES
('New York', 40.7128, -74.0060),
('London',   51.5074,  -0.1278),
('Tokyo',    35.6762, 139.6503);

-- Find all locations within 10000 km of Paris (48.8566, 2.3522)
SELECT name,
  6371 * 2 * ASIN(
    SQRT(
      POW(SIN(RADIANS(lat - 48.8566) / 2), 2) +
      COS(RADIANS(48.8566)) * COS(RADIANS(lat)) *
      POW(SIN(RADIANS(lon - 2.3522) / 2), 2)
    )
  ) AS distance_km
FROM locations
HAVING distance_km < 10000
ORDER BY distance_km;
```

## Using TAN() for Slope Calculations

```sql
-- Calculate slope as rise/run from an angle in degrees
SELECT angle_deg,
  TAN(RADIANS(angle_deg)) AS slope
FROM (
  SELECT 0 AS angle_deg UNION ALL
  SELECT 30 UNION ALL
  SELECT 45 UNION ALL
  SELECT 60
) angles;
```

## Inverse Trigonometric Functions

MySQL also provides the inverse functions:

```sql
SELECT ASIN(1);           -- PI()/2 (90 degrees)
SELECT ACOS(1);           -- 0
SELECT ATAN(1);           -- PI()/4 (45 degrees)
SELECT ATAN2(1, 1);       -- PI()/4 (45 degrees, two-argument form)
```

## Summary

`SIN()`, `COS()`, and `TAN()` are MySQL's core trigonometric functions, all taking angle values in radians. Convert degrees using `RADIANS()` or multiply by `PI()/180`. They are most commonly used for geospatial distance calculations via the Haversine formula and for computing geometric projections directly in SQL without application-layer processing.
