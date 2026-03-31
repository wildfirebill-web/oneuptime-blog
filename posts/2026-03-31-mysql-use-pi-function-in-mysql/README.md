# How to Use PI() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Pi Function, Math Functions, Sql, Geometric Calculations

Description: Learn how to use the PI() function in MySQL to access the value of pi for geometric calculations, circle measurements, and trigonometric formulas.

---

## Introduction

`PI()` is a MySQL mathematical function that returns the value of pi (3.141592653589793). It takes no arguments and is used in geometric calculations involving circles, spheres, angles, and trigonometry. Since MySQL does not have a built-in pi constant, `PI()` provides the standard way to access this mathematical constant in queries.

## Basic Syntax

```sql
SELECT PI();
```

## Basic Example

```sql
SELECT PI();
-- Returns: 3.141592653589793
```

## Calculating Circle Area

Area = pi * r^2

```sql
SELECT
  name,
  radius,
  PI() * POW(radius, 2) AS area
FROM circles;
```

## Calculating Circle Circumference

Circumference = 2 * pi * r

```sql
SELECT
  name,
  radius,
  2 * PI() * radius AS circumference
FROM circles;
```

## Calculating Sphere Volume

Volume = (4/3) * pi * r^3

```sql
SELECT
  name,
  radius,
  (4/3) * PI() * POW(radius, 3) AS volume
FROM spheres;
```

## Converting Degrees to Radians

MySQL trigonometric functions expect radians. Use `RADIANS()` or compute with PI():

```sql
-- Convert 180 degrees to radians
SELECT 180 * PI() / 180;  -- Returns: 3.14159... (= pi)
SELECT RADIANS(180);       -- Equivalent, built-in function

-- Convert 90 degrees to radians
SELECT RADIANS(90);        -- Returns: 1.5707963267948966 (= pi/2)
```

## Converting Radians to Degrees

```sql
SELECT DEGREES(PI());      -- Returns: 180
SELECT PI() * 180 / PI();  -- Returns: 180
```

## Using PI() with Trigonometric Functions

```sql
-- SIN of 90 degrees (pi/2 radians)
SELECT SIN(PI() / 2);     -- Returns: 1

-- COS of 180 degrees (pi radians)
SELECT COS(PI());          -- Returns: -1

-- TAN of 45 degrees (pi/4 radians)
SELECT TAN(PI() / 4);     -- Returns: 1 (approximately)
```

## Haversine Formula for Geographic Distance

Calculate the distance between two geographic points using PI():

```sql
SELECT
  6371 * 2 * ASIN(
    SQRT(
      POW(SIN((lat2 - lat1) * PI() / 180 / 2), 2) +
      COS(lat1 * PI() / 180) * COS(lat2 * PI() / 180) *
      POW(SIN((lon2 - lon1) * PI() / 180 / 2), 2)
    )
  ) AS distance_km
FROM location_pairs;
```

## Sector Area Calculation

Area of a circle sector = (angle / 2pi) * pi * r^2 = angle * r^2 / 2

```sql
-- Calculate sector area given angle in degrees
SELECT
  RADIANS(angle_degrees) * POW(radius, 2) / 2 AS sector_area
FROM sectors;
```

## Stored Function for Circumference

```sql
DELIMITER //
CREATE FUNCTION circle_circumference(r DOUBLE) RETURNS DOUBLE DETERMINISTIC
BEGIN
  RETURN 2 * PI() * r;
END //
DELIMITER ;

SELECT circle_circumference(5);  -- Returns: 31.41592653589793
```

## Practical Example: Finding Nearby Locations

Using a simplified distance formula with PI():

```sql
SELECT
  location_name,
  latitude,
  longitude,
  -- Approximate distance in km from point (40.7128, -74.0060) = NYC
  111.32 * DEGREES(
    ACOS(
      LEAST(1.0,
        COS(RADIANS(40.7128)) * COS(RADIANS(latitude)) *
        COS(RADIANS(longitude) - RADIANS(-74.0060)) +
        SIN(RADIANS(40.7128)) * SIN(RADIANS(latitude))
      )
    )
  ) AS distance_km
FROM locations
HAVING distance_km < 50
ORDER BY distance_km;
```

## PI() Precision

MySQL stores `PI()` as a double-precision floating point:

```sql
SELECT PI();           -- Returns: 3.141592653589793 (standard display)
SELECT FORMAT(PI(), 15); -- Returns: 3.141592653589793
```

## Summary

`PI()` provides the mathematical constant pi in MySQL for geometric and trigonometric calculations. Use it to compute circle areas, circumferences, sphere volumes, and in formulas requiring angle conversions between degrees and radians. Combine it with `SIN()`, `COS()`, `TAN()`, `RADIANS()`, and `DEGREES()` for complete trigonometric workflows.
