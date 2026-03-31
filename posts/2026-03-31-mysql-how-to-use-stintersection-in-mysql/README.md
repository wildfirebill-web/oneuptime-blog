# How to Use ST_Intersection() in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Spatial Functions, GIS, Geometry

Description: Learn how to use ST_Intersection() in MySQL to compute the geometric intersection of two spatial objects with practical examples.

---

## What is ST_Intersection()

`ST_Intersection()` is a MySQL spatial function that returns a geometry representing the shared portion of two input geometries. If the geometries do not overlap, it returns an empty geometry. It is part of MySQL's GIS (Geographic Information System) support and is particularly useful when working with geographic data.

Syntax:

```sql
ST_Intersection(geometry1, geometry2)
```

Both arguments must be valid geometry objects. The function returns a geometry of the appropriate type (point, linestring, polygon, or geometry collection).

## Setting Up a Spatial Table

Before using `ST_Intersection()`, create a table with a spatial column:

```sql
CREATE TABLE regions (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  area POLYGON NOT NULL,
  SPATIAL INDEX(area)
);
```

Insert some overlapping polygons:

```sql
INSERT INTO regions (name, area) VALUES
  ('Region A', ST_GeomFromText('POLYGON((0 0, 0 10, 10 10, 10 0, 0 0))')),
  ('Region B', ST_GeomFromText('POLYGON((5 5, 5 15, 15 15, 15 5, 5 5))'));
```

## Computing the Intersection

Use `ST_Intersection()` to find the overlapping area between two regions:

```sql
SELECT
  ST_AsText(
    ST_Intersection(
      (SELECT area FROM regions WHERE name = 'Region A'),
      (SELECT area FROM regions WHERE name = 'Region B')
    )
  ) AS intersection_area;
```

Expected result:

```text
POLYGON((10 10,10 5,5 5,5 10,10 10))
```

## Checking if Intersection is Empty

Use `ST_IsEmpty()` to confirm whether two geometries actually intersect:

```sql
SELECT
  r1.name AS region1,
  r2.name AS region2,
  ST_IsEmpty(ST_Intersection(r1.area, r2.area)) AS no_overlap
FROM regions r1
JOIN regions r2 ON r1.id < r2.id;
```

## Computing Intersection Area

To calculate the numeric area of the intersection, combine `ST_Intersection()` with `ST_Area()`:

```sql
SELECT
  r1.name AS region1,
  r2.name AS region2,
  ST_Area(ST_Intersection(r1.area, r2.area)) AS overlap_area
FROM regions r1
JOIN regions r2 ON r1.id < r2.id
WHERE NOT ST_IsEmpty(ST_Intersection(r1.area, r2.area));
```

## Using ST_Intersection with Points and Lines

`ST_Intersection()` works with any geometry types:

```sql
-- Intersection of two linestrings
SELECT ST_AsText(
  ST_Intersection(
    ST_GeomFromText('LINESTRING(0 0, 10 10)'),
    ST_GeomFromText('LINESTRING(0 10, 10 0)')
  )
) AS crossing_point;
```

This returns the point where the two lines cross:

```text
POINT(5 5)
```

## Practical Use Case - Finding Common Coverage Areas

A common use case is finding cells covered by two different service zones:

```sql
SELECT
  z1.zone_name AS zone1,
  z2.zone_name AS zone2,
  ST_AsText(ST_Intersection(z1.boundary, z2.boundary)) AS overlap
FROM service_zones z1
JOIN service_zones z2 ON z1.id < z2.id
WHERE ST_Intersects(z1.boundary, z2.boundary);
```

Using `ST_Intersects()` as a pre-filter improves performance by avoiding the more expensive `ST_Intersection()` call on non-overlapping geometries.

## Summary

`ST_Intersection()` in MySQL computes the geometric overlap between two spatial objects and is essential for GIS-based queries. Pair it with `ST_IsEmpty()` to filter empty results, `ST_Area()` to measure overlap, and `ST_Intersects()` as an index-friendly pre-filter. Proper spatial indexes on geometry columns are critical for query performance.
