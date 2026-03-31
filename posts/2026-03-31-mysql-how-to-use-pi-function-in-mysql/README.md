# How to Use PI() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, PI, Math Function, Geometry, SQL

Description: Learn how to use the PI() function in MySQL to access the mathematical constant pi for trigonometric and geometric calculations in your queries.

---

## What Is the PI() Function?

`PI()` returns the value of the mathematical constant pi (approximately 3.141592653589793). It takes no arguments and is used in trigonometric calculations, circle geometry, and any formula that requires pi. MySQL stores it as a double-precision floating-point number.

```sql
SELECT PI();
-- Returns: 3.141592653589793
```

## Basic Usage

```sql
-- PI() returns the constant
SELECT PI();               -- 3.141592653589793

-- Use in expressions
SELECT PI() * 2;           -- 6.283185307179586 (2*pi, one full revolution in radians)
SELECT PI() / 2;           -- 1.5707963267948966 (pi/2, 90 degrees in radians)
SELECT PI() / 180;         -- 0.017453292519943295 (degrees to radians conversion factor)
SELECT 180 / PI();         -- 57.29577951308232 (radians to degrees conversion factor)
```

## Circle Calculations

```sql
-- Area of a circle: A = pi * r^2
SELECT
    radius,
    ROUND(PI() * POW(radius, 2), 4) AS area
FROM circle_specs;

-- Circumference: C = 2 * pi * r
SELECT
    radius,
    ROUND(2 * PI() * radius, 4) AS circumference
FROM circle_specs;

-- Diameter from radius
SELECT
    radius,
    radius * 2 AS diameter,
    ROUND(PI() * radius * 2, 4) AS circumference_from_diameter
FROM circle_specs;

-- Example with literal value
SELECT
    5 AS radius,
    ROUND(PI() * POW(5, 2), 2) AS area,        -- 78.54
    ROUND(2 * PI() * 5, 2) AS circumference;    -- 31.42
```

## Sphere and 3D Geometry

```sql
-- Volume of a sphere: V = (4/3) * pi * r^3
SELECT
    radius,
    ROUND((4.0/3.0) * PI() * POW(radius, 3), 4) AS volume
FROM sphere_data;

-- Surface area of a sphere: SA = 4 * pi * r^2
SELECT
    radius,
    ROUND(4 * PI() * POW(radius, 2), 4) AS surface_area
FROM sphere_data;
```

## Trigonometric Functions with PI

MySQL's trig functions work in radians, so PI() is essential for converting:

```sql
-- Convert degrees to radians: radians = degrees * (PI / 180)
SELECT
    degrees,
    ROUND(degrees * PI() / 180, 6) AS radians
FROM angle_table;

-- Convert radians to degrees: degrees = radians * (180 / PI)
SELECT
    radians,
    ROUND(radians * 180 / PI(), 4) AS degrees
FROM radian_table;

-- Trig functions using PI
SELECT
    ROUND(SIN(PI() / 6), 4) AS sin_30deg,   -- 0.5
    ROUND(COS(PI() / 3), 4) AS cos_60deg,   -- 0.5
    ROUND(TAN(PI() / 4), 4) AS tan_45deg;   -- 1.0
```

## Geographic Distance Calculation (Haversine Formula)

```sql
-- Calculate distance between two geographic coordinates using PI
-- Simplified Haversine formula for illustration
SELECT
    lat1, lon1, lat2, lon2,
    ROUND(
        6371 * 2 * ASIN(SQRT(
            POW(SIN((lat2 - lat1) * PI() / 360), 2) +
            COS(lat1 * PI() / 180) * COS(lat2 * PI() / 180) *
            POW(SIN((lon2 - lon1) * PI() / 360), 2)
        )),
        2
    ) AS distance_km
FROM location_pairs;
```

## Practical Example - Circular Coverage Zones

```sql
CREATE TABLE delivery_zones (
    id INT AUTO_INCREMENT PRIMARY KEY,
    center_lat DECIMAL(9, 6),
    center_lon DECIMAL(9, 6),
    radius_km DECIMAL(6, 2),
    zone_name VARCHAR(100)
);

-- Calculate area of each delivery zone in square kilometers
SELECT
    zone_name,
    radius_km,
    ROUND(PI() * POW(radius_km, 2), 2) AS area_sq_km
FROM delivery_zones;

-- Calculate the bounding box delta for a circular zone
SELECT
    zone_name,
    radius_km,
    ROUND(radius_km / 111.32, 4) AS lat_delta_degrees,
    ROUND(radius_km / (111.32 * COS(center_lat * PI() / 180)), 4) AS lon_delta_degrees
FROM delivery_zones;
```

## Using PI in Custom Functions

```sql
DELIMITER //
CREATE FUNCTION circle_area(radius DOUBLE)
RETURNS DOUBLE DETERMINISTIC
BEGIN
    RETURN PI() * POW(radius, 2);
END //

CREATE FUNCTION degrees_to_radians(deg DOUBLE)
RETURNS DOUBLE DETERMINISTIC
BEGIN
    RETURN deg * PI() / 180;
END //
DELIMITER ;

-- Use the functions
SELECT circle_area(7);                -- 153.9380...
SELECT degrees_to_radians(90);        -- 1.5707...
```

## Summary

`PI()` provides the mathematical constant pi (3.141592653589793) in MySQL for use in geometric and trigonometric calculations. Use it for computing circle areas and circumferences, sphere volumes, angle conversions between degrees and radians, and geographic distance calculations. Since MySQL's trigonometric functions (`SIN`, `COS`, `TAN`) operate in radians, `PI()` is the essential bridge for degree-based angle inputs.
