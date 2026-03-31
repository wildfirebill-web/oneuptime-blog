# How to Use ASIN(), ACOS(), ATAN() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Math Function, Database

Description: Learn how MySQL's ASIN(), ACOS(), and ATAN() functions compute inverse trigonometric values in radians, with practical geometry and GIS examples.

---

## What are ASIN(), ACOS(), and ATAN()?

MySQL provides three inverse trigonometric functions that return the angle (in radians) whose trigonometric ratio matches the input:

- `ASIN(X)` - arc sine: returns the angle whose sine is X. Domain: [-1, 1]. Range: [-pi/2, pi/2]
- `ACOS(X)` - arc cosine: returns the angle whose cosine is X. Domain: [-1, 1]. Range: [0, pi]
- `ATAN(X)` - arc tangent: returns the angle whose tangent is X. Domain: all reals. Range: (-pi/2, pi/2)
- `ATAN(Y, X)` - two-argument arc tangent (also written `ATAN2(Y, X)`): returns the angle of the vector (X, Y). Range: (-pi, pi]

All results are in radians. Use `DEGREES()` to convert to degrees.

## Basic Examples

```sql
SELECT ASIN(1);
-- Result: 1.5707963267948966  (pi/2 = 90 degrees)

SELECT ASIN(0.5);
-- Result: 0.5235987755982988  (30 degrees)

SELECT ACOS(1);
-- Result: 0

SELECT ACOS(0);
-- Result: 1.5707963267948966  (90 degrees)

SELECT ATAN(1);
-- Result: 0.7853981633974483  (45 degrees)

SELECT DEGREES(ATAN(1));
-- Result: 45
```

## Converting Results to Degrees

```sql
SELECT DEGREES(ASIN(0.5)) AS angle_deg;
-- Result: 30

SELECT DEGREES(ACOS(0.5)) AS angle_deg;
-- Result: 60

SELECT DEGREES(ATAN(1)) AS angle_deg;
-- Result: 45
```

## Domain Validation - Out-of-Range Inputs

`ASIN()` and `ACOS()` return `NULL` for inputs outside [-1, 1]:

```sql
SELECT ASIN(2);
-- Result: NULL  (out of domain)

SELECT ACOS(-2);
-- Result: NULL
```

`ATAN()` has no domain restrictions:

```sql
SELECT ATAN(1000000);
-- Result: 1.5707953... (approaches pi/2)
```

## Haversine Formula for Geographic Distance

The haversine formula uses `ASIN()` to compute the great-circle distance between two geographic coordinates:

```sql
-- Distance in kilometers between two lat/lon points
SELECT
  6371 * 2 * ASIN(
    SQRT(
      POWER(SIN(RADIANS(lat2 - lat1) / 2), 2) +
      COS(RADIANS(lat1)) * COS(RADIANS(lat2)) *
      POWER(SIN(RADIANS(lon2 - lon1) / 2), 2)
    )
  ) AS distance_km
FROM location_pairs;
```

## Finding Nearby Locations

```sql
SELECT
  id,
  name,
  lat,
  lon,
  6371 * 2 * ASIN(
    SQRT(
      POWER(SIN(RADIANS(lat - 40.7128) / 2), 2) +
      COS(RADIANS(40.7128)) * COS(RADIANS(lat)) *
      POWER(SIN(RADIANS(lon - (-74.0060)) / 2), 2)
    )
  ) AS km_from_nyc
FROM locations
HAVING km_from_nyc < 50
ORDER BY km_from_nyc;
```

## Two-Argument ATAN() for Bearing

`ATAN(Y, X)` computes the bearing angle of a vector:

```sql
SELECT DEGREES(ATAN(3, 4)) AS bearing_deg;
-- Result: 36.87  (angle of vector (4,3))
```

In navigation, compute the bearing from point A to point B:

```sql
SELECT
  DEGREES(
    ATAN(
      SIN(RADIANS(lon_b - lon_a)) * COS(RADIANS(lat_b)),
      COS(RADIANS(lat_a)) * SIN(RADIANS(lat_b)) -
      SIN(RADIANS(lat_a)) * COS(RADIANS(lat_b)) * COS(RADIANS(lon_b - lon_a))
    )
  ) AS bearing_degrees
FROM routes;
```

## Computing Angle of Triangle Sides

Given three sides a, b, c, compute an angle using the law of cosines:

```sql
-- angle C opposite side c
SELECT DEGREES(ACOS((a*a + b*b - c*c) / (2*a*b))) AS angle_C
FROM triangles;
```

## Summary

`ASIN()`, `ACOS()`, and `ATAN()` return inverse trigonometric angles in radians. `ASIN()` and `ACOS()` require inputs in [-1, 1] and return `NULL` outside that range. `ATAN()` accepts any real number; its two-argument form `ATAN(Y, X)` handles quadrant-correct bearing calculation. Use `DEGREES()` to convert results. The most common MySQL application is the haversine formula for computing geographic distances between latitude/longitude coordinates.
