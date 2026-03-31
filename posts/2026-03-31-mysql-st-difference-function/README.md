# How to Use ST_Difference() in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Spatial, Geometry, Function, GIS

Description: Learn how to use MySQL's ST_Difference() function to compute the geometric difference between two geometries - the area in one but not the other.

---

## What is ST_Difference()?

`ST_Difference(g1, g2)` returns the part of geometry `g1` that does not intersect with geometry `g2`. It is the set-theoretic difference operation: g1 minus the overlap with g2. If the two geometries do not overlap at all, the result is `g1` unchanged.

This is the spatial equivalent of subtracting one shape from another.

## Basic Syntax

```sql
ST_Difference(geometry1, geometry2)
```

Both arguments must be in the same SRID. The result is a geometry in that same SRID.

## Simple Example: Polygon Difference

```sql
-- Square with a rectangular chunk removed
SELECT ST_AsText(
  ST_Difference(
    ST_GeomFromText('POLYGON((0 0, 4 0, 4 4, 0 4, 0 0))', 0),
    ST_GeomFromText('POLYGON((2 0, 4 0, 4 4, 2 4, 2 0))', 0)
  )
) AS result;
```

```text
+--------------------------------------+
| result                               |
+--------------------------------------+
| POLYGON((0 0,2 0,2 4,0 4,0 0))      |
+--------------------------------------+
```

The right half of the original square was subtracted, leaving the left half.

## Practical Example: Excluded Zones

A delivery service might need to find the serviceable area within a city after removing restricted zones:

```sql
CREATE TABLE service_areas (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100),
  region POLYGON NOT NULL SRID 0,
  SPATIAL INDEX (region)
);

INSERT INTO service_areas (name, region) VALUES
  ('Full City',
    ST_GeomFromText('POLYGON((0 0, 100 0, 100 100, 0 100, 0 0))', 0)),
  ('Airport No-Fly Zone',
    ST_GeomFromText('POLYGON((60 60, 100 60, 100 100, 60 100, 60 60))', 0));

-- Compute deliverable area
SELECT ST_AsText(
  ST_Difference(
    (SELECT region FROM service_areas WHERE name = 'Full City'),
    (SELECT region FROM service_areas WHERE name = 'Airport No-Fly Zone')
  )
) AS deliverable_area;
```

## Checking the Remaining Area

Combine with `ST_Area()` to quantify how much area remains after the subtraction:

```sql
SELECT
  ST_Area(city.region) AS original_area,
  ST_Area(ST_Difference(city.region, restricted.region)) AS remaining_area
FROM
  (SELECT region FROM service_areas WHERE name = 'Full City') AS city,
  (SELECT region FROM service_areas WHERE name = 'Airport No-Fly Zone') AS restricted;
```

## Non-overlapping Geometries

If the two geometries do not intersect, `ST_Difference()` returns g1 unchanged:

```sql
SELECT ST_AsText(
  ST_Difference(
    ST_GeomFromText('POLYGON((0 0, 2 0, 2 2, 0 2, 0 0))', 0),
    ST_GeomFromText('POLYGON((5 5, 7 5, 7 7, 5 7, 5 5))', 0)
  )
) AS result;
-- Returns the original polygon unchanged
```

## Symmetry with ST_Union and ST_Intersection

The three set operations are related:

- `ST_Union(g1, g2)` - everything in either g1 or g2
- `ST_Intersection(g1, g2)` - only what is in both
- `ST_Difference(g1, g2)` - g1 minus what is in both

## SRID Requirements

Both geometries must share the same SRID. Using mismatched SRIDs raises an error:

```sql
-- This will raise an error
SELECT ST_Difference(
  ST_GeomFromText('POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))', 0),
  ST_GeomFromText('POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))', 4326)
);
```

Use `ST_Transform()` to convert to a common SRID before calling `ST_Difference()`.

## Summary

`ST_Difference(g1, g2)` returns the portion of g1 that does not overlap with g2. Use it to compute available areas after excluding restricted zones, calculate net territory coverage, or perform any spatial subtraction operation. Always ensure both input geometries share the same SRID.
