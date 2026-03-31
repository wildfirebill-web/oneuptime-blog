# How to Use SIN(), COS(), TAN() Trigonometric Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Trigonometry, Sin Cos Tan, Math Functions, Sql

Description: Learn how to use SIN(), COS(), and TAN() trigonometric functions in MySQL with examples for angle calculations, geometric computations, and geographic distance.

---

## Introduction

MySQL provides trigonometric functions `SIN()`, `COS()`, and `TAN()` for sine, cosine, and tangent calculations respectively. All these functions accept angles in radians. MySQL also provides `ASIN()`, `ACOS()`, `ATAN()`, and `ATAN2()` as inverse functions, along with `DEGREES()` and `RADIANS()` for angle conversion.

## Basic Syntax

```sql
SIN(angle_in_radians)
COS(angle_in_radians)
TAN(angle_in_radians)
```

## Converting Degrees to Radians

MySQL trig functions require radians. Use `RADIANS()` to convert:

```sql
SELECT RADIANS(0);    -- Returns: 0
SELECT RADIANS(90);   -- Returns: 1.5707963267948966  (pi/2)
SELECT RADIANS(180);  -- Returns: 3.141592653589793   (pi)
SELECT RADIANS(360);  -- Returns: 6.283185307179586   (2*pi)
```

## SIN() Examples

```sql
SELECT SIN(0);               -- Returns: 0
SELECT SIN(RADIANS(30));     -- Returns: 0.5  (sin 30° = 0.5)
SELECT SIN(RADIANS(90));     -- Returns: 1    (sin 90° = 1)
SELECT SIN(RADIANS(45));     -- Returns: 0.7071... (sin 45° = sqrt(2)/2)
SELECT SIN(PI() / 6);        -- Returns: 0.5  (pi/6 = 30°)
```

## COS() Examples

```sql
SELECT COS(0);               -- Returns: 1
SELECT COS(RADIANS(60));     -- Returns: 0.5  (cos 60° = 0.5)
SELECT COS(RADIANS(90));     -- Returns: ~0   (cos 90° = 0)
SELECT COS(RADIANS(0));      -- Returns: 1    (cos 0° = 1)
SELECT COS(PI());            -- Returns: -1   (cos 180° = -1)
```

## TAN() Examples

```sql
SELECT TAN(0);               -- Returns: 0
SELECT TAN(RADIANS(45));     -- Returns: 1    (tan 45° = 1)
SELECT TAN(RADIANS(0));      -- Returns: 0
-- TAN(90°) is undefined (infinity) - returns a very large number
SELECT TAN(RADIANS(89.99));  -- Returns: very large number
```

## Inverse Functions

```sql
SELECT ASIN(0.5);    -- Returns: 0.5235...  (= pi/6 = 30 degrees in radians)
SELECT DEGREES(ASIN(0.5));  -- Returns: 30

SELECT ACOS(0.5);    -- Returns: 1.0471...  (= pi/3 = 60 degrees in radians)
SELECT DEGREES(ACOS(0.5));  -- Returns: 60

SELECT ATAN(1);      -- Returns: 0.7853...  (= pi/4 = 45 degrees in radians)
SELECT DEGREES(ATAN(1));    -- Returns: 45
```

## Pythagorean Identity Verification

```sql
-- SIN^2 + COS^2 = 1 for any angle
SELECT POW(SIN(RADIANS(37)), 2) + POW(COS(RADIANS(37)), 2) AS identity;
-- Returns: 1 (or very close to 1 due to floating point)
```

## Calculating Triangle Sides

Given an angle and one side, find another side:

```sql
-- In a right triangle with hypotenuse=10 and angle=30 degrees:
-- opposite side = hypotenuse * SIN(angle)
-- adjacent side = hypotenuse * COS(angle)
SELECT
  10 * SIN(RADIANS(30)) AS opposite_side,  -- Returns: 5
  10 * COS(RADIANS(30)) AS adjacent_side;  -- Returns: 8.66
```

## Haversine Formula for Geographic Distance

Calculate distance between two GPS coordinates:

```sql
SELECT
  p1.name AS location_a,
  p2.name AS location_b,
  6371 * ACOS(
    LEAST(1.0,
      COS(RADIANS(p1.latitude)) *
      COS(RADIANS(p2.latitude)) *
      COS(RADIANS(p2.longitude) - RADIANS(p1.longitude)) +
      SIN(RADIANS(p1.latitude)) *
      SIN(RADIANS(p2.latitude))
    )
  ) AS distance_km
FROM locations p1
JOIN locations p2 ON p1.id < p2.id
WHERE p1.id = 1 AND p2.id = 2;
```

## Wave Pattern Generation

Generate a sine wave for data analysis or visualization:

```sql
SELECT
  n AS x_position,
  SIN(n * PI() / 180) AS y_value
FROM (
  SELECT @n := @n + 1 AS n
  FROM information_schema.columns
  CROSS JOIN (SELECT @n := -1) AS init
  LIMIT 361
) AS series;
```

## ATAN2() for Angle Between Two Points

```sql
-- Angle from origin to point (x, y)
SELECT DEGREES(ATAN2(y, x)) AS angle_degrees
FROM points;
```

## Stored Function: Haversine Distance

```sql
DELIMITER //
CREATE FUNCTION haversine_km(lat1 DOUBLE, lon1 DOUBLE, lat2 DOUBLE, lon2 DOUBLE)
RETURNS DOUBLE DETERMINISTIC
BEGIN
  RETURN 6371 * ACOS(
    LEAST(1.0,
      COS(RADIANS(lat1)) * COS(RADIANS(lat2)) *
      COS(RADIANS(lon2) - RADIANS(lon1)) +
      SIN(RADIANS(lat1)) * SIN(RADIANS(lat2))
    )
  );
END //
DELIMITER ;

SELECT haversine_km(40.7128, -74.0060, 34.0522, -118.2437) AS ny_to_la_km;
-- Returns approximately 3940 km
```

## Summary

`SIN()`, `COS()`, and `TAN()` are MySQL's core trigonometric functions, all accepting angles in radians. Use `RADIANS()` to convert from degrees before calling them, and `DEGREES()` to convert results back. The inverse functions `ASIN()`, `ACOS()`, and `ATAN()` compute angles from values. These functions are essential for geographic distance calculations, geometric computations, and scientific data analysis.
