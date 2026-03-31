# How to Use DEGREES() and RADIANS() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Numeric Function, Trigonometry, Database

Description: Learn how to use MySQL DEGREES() and RADIANS() to convert angles between degrees and radians for trigonometric function compatibility.

---

## Overview

MySQL trigonometric functions (`SIN()`, `COS()`, `TAN()`, `ASIN()`, `ACOS()`, `ATAN()`) work exclusively in **radians**. MySQL provides two conversion functions to move between the degree and radian scales:

- `RADIANS(X)` - converts degrees to radians.
- `DEGREES(X)` - converts radians to degrees.

Understanding both is essential for working with angle-based data that may be stored or presented in degrees.

---

## RADIANS() Function

Converts an angle from degrees to radians.

**Syntax:**

```sql
RADIANS(X)
```

**Formula:** `radians = X * PI() / 180`

```sql
SELECT RADIANS(180);
-- Returns: 3.141592653589793  (PI)

SELECT RADIANS(90);
-- Returns: 1.5707963267948966 (PI/2)

SELECT RADIANS(45);
-- Returns: 0.7853981633974483 (PI/4)

SELECT RADIANS(0);
-- Returns: 0.0

SELECT RADIANS(360);
-- Returns: 6.283185307179586  (2*PI)

SELECT RADIANS(NULL);
-- Returns: NULL
```

---

## DEGREES() Function

Converts an angle from radians to degrees.

**Syntax:**

```sql
DEGREES(X)
```

**Formula:** `degrees = X * 180 / PI()`

```sql
SELECT DEGREES(PI());
-- Returns: 180.0

SELECT DEGREES(PI() / 2);
-- Returns: 90.0

SELECT DEGREES(PI() / 4);
-- Returns: 45.0

SELECT DEGREES(2 * PI());
-- Returns: 360.0

SELECT DEGREES(0);
-- Returns: 0.0

SELECT DEGREES(1);
-- Returns: 57.29577951308232  (1 radian in degrees)

SELECT DEGREES(NULL);
-- Returns: NULL
```

---

## How DEGREES() and RADIANS() Relate

```mermaid
flowchart LR
    A[Angle in Degrees] -->|"RADIANS(d) = d * PI/180"| B[Angle in Radians]
    B -->|"DEGREES(r) = r * 180/PI"| A
    B --> C["SIN() / COS() / TAN()"]
    C -->|"ASIN() / ACOS() / ATAN()"| D[Radians output]
    D -->|DEGREES()| E[Degrees output]
```

---

## Round-Trip Conversion

```sql
-- Degrees -> Radians -> Degrees
SELECT DEGREES(RADIANS(90));
-- Returns: 90.0

SELECT DEGREES(RADIANS(360));
-- Returns: 360.0

-- Radians -> Degrees -> Radians
SELECT RADIANS(DEGREES(PI()));
-- Returns: 3.141592653589793
```

---

## Converting Trig Results Back to Degrees

Inverse trigonometric functions return results in radians. Use `DEGREES()` to convert back:

```sql
-- What angle has sin = 0.5?
SELECT DEGREES(ASIN(0.5));
-- Returns: 30.0

-- What angle has cos = 0.5?
SELECT DEGREES(ACOS(0.5));
-- Returns: 60.0

-- What angle has tan = 1?
SELECT DEGREES(ATAN(1));
-- Returns: 45.0

-- What angle has tan = opposite/adjacent = 3/4 ?
SELECT DEGREES(ATAN2(3, 4));
-- Returns: 36.86989764584402
```

---

## Practical: Bearing Calculation Between Two GPS Points

```sql
-- Calculate bearing (heading) from point A to point B in degrees
SET @lat1 = RADIANS(51.5074);  -- London
SET @lon1 = RADIANS(-0.1278);
SET @lat2 = RADIANS(48.8566);  -- Paris
SET @lon2 = RADIANS(2.3522);

SELECT ROUND(
    MOD(
        DEGREES(
            ATAN2(
                SIN(@lon2 - @lon1) * COS(@lat2),
                COS(@lat1) * SIN(@lat2) - SIN(@lat1) * COS(@lat2) * COS(@lon2 - @lon1)
            )
        ) + 360,
        360
    ), 1
) AS bearing_degrees;
-- Returns: approximate compass bearing from London to Paris
```

---

## Working with Stored Degree Data

When latitude and longitude are stored as degrees (common practice), always convert with `RADIANS()` before trig operations:

```sql
CREATE TABLE waypoints (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    latitude DOUBLE,   -- stored in degrees
    longitude DOUBLE   -- stored in degrees
);

INSERT INTO waypoints VALUES
(1, 'London', 51.5074, -0.1278),
(2, 'Paris',  48.8566,  2.3522),
(3, 'Berlin', 52.5200, 13.4050);

-- Distance from London to each other waypoint (Haversine)
SELECT
    w.name,
    ROUND(
        6371 * 2 * ASIN(SQRT(
            POWER(SIN(RADIANS(w.latitude  - h.latitude)  / 2), 2) +
            COS(RADIANS(h.latitude)) * COS(RADIANS(w.latitude)) *
            POWER(SIN(RADIANS(w.longitude - h.longitude) / 2), 2)
        )), 1
    ) AS dist_km
FROM waypoints w
CROSS JOIN waypoints h WHERE h.name = 'London' AND w.name != 'London';
```

---

## Angle Normalization

```sql
-- Normalize an angle to the range [0, 360)
SELECT MOD(DEGREES(ATAN2(-1, -1)) + 360, 360) AS normalized_angle;
-- ATAN2(-1,-1) = -135 degrees -> normalized to 225
```

---

## DEGREES() for Readable Output

When performing geometric analysis, convert radian results to degrees for user-facing output:

```sql
-- Table of common angles
SELECT
    angle_deg,
    ROUND(RADIANS(angle_deg), 4) AS radians,
    ROUND(SIN(RADIANS(angle_deg)), 4) AS sin_val,
    ROUND(COS(RADIANS(angle_deg)), 4) AS cos_val
FROM (
    SELECT 0  AS angle_deg UNION ALL SELECT 30 UNION ALL SELECT 45
    UNION ALL SELECT 60 UNION ALL SELECT 90  UNION ALL SELECT 180
) t;
```

---

## Summary

`RADIANS()` and `DEGREES()` handle angle unit conversion in MySQL. `RADIANS()` converts degrees to radians (multiply by PI/180), and `DEGREES()` converts radians to degrees (multiply by 180/PI). Since all MySQL trigonometric functions operate in radians, use `RADIANS()` to prepare degree-based inputs and `DEGREES()` to convert inverse trig function outputs back to degrees for human-readable results. They are essential for geographic distance and bearing calculations where coordinates are stored in degrees.
