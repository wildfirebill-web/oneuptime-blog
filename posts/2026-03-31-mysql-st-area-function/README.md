# How to Use ST_Area() in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Spatial, Geometry, Function, GIS

Description: Learn how to use the MySQL ST_Area() function to calculate the area of polygon geometries in both Cartesian and geodetic coordinate systems.

---

## What is ST_Area()?

`ST_Area()` calculates the area of a polygon or multipolygon geometry. The function was enhanced in MySQL 8.0 to support geodetic calculations using proper SRID-based coordinate systems. For geographic coordinates (latitude/longitude), the result is expressed in square meters when using SRID 4326.

## Basic Syntax

```sql
ST_Area(geometry [, unit])
```

The optional `unit` parameter (MySQL 8.0.24+) specifies the unit of measurement: `'square_metre'`, `'square_kilometre'`, `'square_foot'`, etc.

## Calculating Area of a Polygon

```sql
-- Cartesian (SRID 0) - area in the same units as coordinates
SELECT ST_Area(
  ST_GeomFromText('POLYGON((0 0, 4 0, 4 3, 0 3, 0 0))', 0)
) AS area;
```

```text
+------+
| area |
+------+
|   12 |
+------+
```

A 4x3 rectangle has an area of 12 square units.

## Geodetic Area with SRID 4326

For real-world geographic data stored with SRID 4326 (WGS 84), `ST_Area()` returns area in square meters:

```sql
-- Area of a geographic polygon
SELECT ST_Area(
  ST_GeomFromText(
    'POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))',
    4326
  )
) AS area_sq_meters;
```

Note: In SRID 4326, coordinates are (latitude, longitude) order. Always be consistent with axis order.

## Using Unit Parameter

```sql
-- Return area in square kilometres
SELECT ST_Area(
  ST_GeomFromText('POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))', 4326),
  'square_kilometre'
) AS area_km2;
```

## Practical Example: Zone Area Calculation

```sql
CREATE TABLE zones (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100),
  boundary POLYGON NOT NULL SRID 4326,
  SPATIAL INDEX (boundary)
);

INSERT INTO zones (name, boundary) VALUES (
  'Downtown',
  ST_GeomFromText(
    'POLYGON((40.7128 -74.0060, 40.7180 -74.0060,
              40.7180 -73.9980, 40.7128 -73.9980,
              40.7128 -74.0060))',
    4326
  )
);

SELECT
  name,
  ST_Area(boundary) AS area_sq_m,
  ST_Area(boundary) / 1000000 AS area_sq_km
FROM zones;
```

## Area of a MultiPolygon

`ST_Area()` sums the areas of all polygons in a multipolygon:

```sql
SELECT ST_Area(
  ST_GeomFromText(
    'MULTIPOLYGON(((0 0, 2 0, 2 2, 0 2, 0 0)),
                  ((3 3, 5 3, 5 5, 3 5, 3 3)))',
    0
  )
) AS total_area;
```

```text
+------------+
| total_area |
+------------+
|          8 |
+------------+
```

Two 2x2 squares equal a total area of 8.

## Null Handling

`ST_Area()` returns `NULL` for non-polygon geometries like points and linestrings:

```sql
SELECT ST_Area(ST_GeomFromText('POINT(0 0)', 0));
-- Returns NULL
```

## Summary

`ST_Area()` computes the area of polygon and multipolygon geometries. For Cartesian data (SRID 0), the result is in coordinate units squared. For geographic data (SRID 4326), the result is in square meters by default, or you can specify a unit explicitly. Use `ST_Area()` for zone analysis, territory calculations, and any workflow that requires measuring the size of geographic regions.
