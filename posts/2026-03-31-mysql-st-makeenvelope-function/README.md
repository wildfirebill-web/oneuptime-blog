# How to Use ST_MakeEnvelope() in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Spatial, Geometry, Function, GIS

Description: Learn how to use MySQL's ST_MakeEnvelope() function to create a rectangular bounding box polygon from two corner points for spatial queries.

---

## What is ST_MakeEnvelope()?

`ST_MakeEnvelope(pt1, pt2)` constructs a rectangular polygon from two corner points. This is the most convenient way to define a bounding box for spatial range queries - for example, finding all points within a visible map viewport or a rectangular search area.

It was introduced in MySQL 8.0.22 as a simpler alternative to manually constructing bounding box polygons.

## Basic Syntax

```sql
ST_MakeEnvelope(point1, point2)
```

Both arguments must be `POINT` geometries with the same SRID. Returns a `POLYGON` with five points forming the rectangle.

## Creating a Simple Bounding Box

```sql
SELECT ST_AsText(
  ST_MakeEnvelope(
    ST_GeomFromText('POINT(0 0)', 0),
    ST_GeomFromText('POINT(10 5)', 0)
  )
) AS envelope;
```

```text
+------------------------------------------+
| envelope                                 |
+------------------------------------------+
| POLYGON((0 0,10 0,10 5,0 5,0 0))        |
+------------------------------------------+
```

## Practical Example: Map Viewport Query

A map application needs to find all points within the current viewport (defined by the southwest and northeast corners):

```sql
CREATE TABLE pois (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100),
  location POINT NOT NULL SRID 4326,
  SPATIAL INDEX (location)
);

INSERT INTO pois (name, location) VALUES
  ('Central Park', ST_GeomFromText('POINT(40.7851 -73.9683)', 4326)),
  ('Times Square', ST_GeomFromText('POINT(40.7580 -73.9855)', 4326)),
  ('Brooklyn Bridge', ST_GeomFromText('POINT(40.7061 -73.9969)', 4326));

-- Find POIs within a map viewport
SET @sw_lat = 40.70;
SET @sw_lon = -74.01;
SET @ne_lat = 40.80;
SET @ne_lon = -73.96;

SELECT name
FROM pois
WHERE ST_Within(
  location,
  ST_MakeEnvelope(
    ST_GeomFromText(CONCAT('POINT(', @sw_lat, ' ', @sw_lon, ')'), 4326),
    ST_GeomFromText(CONCAT('POINT(', @ne_lat, ' ', @ne_lon, ')'), 4326)
  )
);
```

## Using ST_MakeEnvelope with MBRContains

For index-accelerated bounding box queries, combine with `MBRContains()`:

```sql
SELECT name
FROM pois
WHERE MBRContains(
  ST_MakeEnvelope(
    ST_GeomFromText('POINT(40.70 -74.01)', 4326),
    ST_GeomFromText('POINT(40.80 -73.96)', 4326)
  ),
  location
);
```

`MBRContains()` uses the spatial R-tree index efficiently, making this approach much faster for large tables.

## ST_MakeEnvelope vs Manual Polygon Construction

Before `ST_MakeEnvelope()`, you had to construct bounding boxes manually:

```sql
-- Old approach: verbose polygon construction
ST_GeomFromText('POLYGON((0 0, 10 0, 10 5, 0 5, 0 0))', 0)

-- New approach: ST_MakeEnvelope
ST_MakeEnvelope(
  ST_GeomFromText('POINT(0 0)', 0),
  ST_GeomFromText('POINT(10 5)', 0)
)
```

Both produce identical results, but `ST_MakeEnvelope()` is cleaner and less error-prone.

## SRID Requirement

Both points must share the same SRID:

```sql
-- This raises an error: SRID mismatch
SELECT ST_MakeEnvelope(
  ST_GeomFromText('POINT(0 0)', 0),
  ST_GeomFromText('POINT(10 5)', 4326)
);
```

## Summary

`ST_MakeEnvelope()` creates a rectangular bounding box polygon from two corner points. It is the most readable and concise way to define spatial search rectangles in MySQL. Use it with `ST_Within()` for point-in-rectangle queries or with `MBRContains()` to leverage the spatial R-tree index for high-performance viewport and bounding box searches.
