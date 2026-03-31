# How to Use ST_Transform() in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Spatial, Geometry, Function, SRID

Description: Learn how to use MySQL's ST_Transform() function to reproject geometry coordinates from one spatial reference system to another.

---

## What is ST_Transform()?

`ST_Transform(geometry, target_srid)` converts a geometry's coordinates from its current spatial reference system (SRS) to the target SRS. Unlike `ST_SRID()` which only relabels the SRID, `ST_Transform()` mathematically reprojects each coordinate.

This is essential when working with data from multiple sources that use different coordinate systems - for example, combining GPS data (SRID 4326) with local survey data in a projected metric system.

## Basic Syntax

```sql
ST_Transform(geometry, target_srid)
```

Both the input SRID and the target SRID must exist in the `INFORMATION_SCHEMA.ST_SPATIAL_REFERENCE_SYSTEMS` table.

## Converting from SRID 4326 to a Projected System

SRID 4326 uses degrees. For accurate distance and area calculations in meters, convert to a projected system like SRID 3857 (Web Mercator used by Google Maps and OpenStreetMap):

```sql
-- Convert a point from geographic (degrees) to Web Mercator (meters)
SELECT ST_AsText(
  ST_Transform(
    ST_GeomFromText('POINT(40.7128 -74.0060)', 4326),
    3857
  )
) AS projected_point;
```

```text
+---------------------------------------+
| projected_point                       |
+---------------------------------------+
| POINT(-8238310.2 4970071.6)           |
+---------------------------------------+
```

The output coordinates are now in meters relative to the Web Mercator origin.

## Converting Back to Geographic Coordinates

```sql
SELECT ST_AsText(
  ST_Transform(
    ST_GeomFromText('POINT(-8238310.2 4970071.6)', 3857),
    4326
  )
) AS geographic_point;
```

## Practical Example: Computing Accurate Distances

Distances computed in SRID 4326 (degrees) are not accurate for long distances. Convert to a projected system first:

```sql
-- Distance in degrees (not useful)
SELECT ST_Distance(
  ST_GeomFromText('POINT(40.7128 -74.0060)', 4326),
  ST_GeomFromText('POINT(40.7580 -73.9855)', 4326)
) AS dist_degrees;

-- Distance in meters using ST_Distance_Sphere (for geodetic)
SELECT ST_Distance_Sphere(
  ST_GeomFromText('POINT(40.7128 -74.0060)', 4326),
  ST_GeomFromText('POINT(40.7580 -73.9855)', 4326)
) AS dist_meters;

-- Distance after projecting to Web Mercator
SELECT ST_Distance(
  ST_Transform(ST_GeomFromText('POINT(40.7128 -74.0060)', 4326), 3857),
  ST_Transform(ST_GeomFromText('POINT(40.7580 -73.9855)', 4326), 3857)
) AS dist_projected_meters;
```

## Checking Available SRIDs

```sql
SELECT SRS_ID, SRS_NAME
FROM INFORMATION_SCHEMA.ST_SPATIAL_REFERENCE_SYSTEMS
WHERE SRS_ID IN (4326, 3857, 3395)
ORDER BY SRS_ID;
```

## Storing Pre-Transformed Data

For high-performance applications, pre-transform and store coordinates in the most useful projection:

```sql
ALTER TABLE locations ADD COLUMN coords_3857 POINT SRID 3857;

UPDATE locations
SET coords_3857 = ST_Transform(coords, 3857);

CREATE SPATIAL INDEX idx_coords_3857 ON locations (coords_3857);
```

## Common SRID Reference

| SRID | Name | Use Case |
|---|---|---|
| 4326 | WGS 84 | GPS, GeoJSON, worldwide data |
| 3857 | Web Mercator | Web maps, tile services |
| 3395 | World Mercator | Older mapping systems |

## Summary

`ST_Transform()` reprojects geometry coordinates from one spatial reference system to another. Use it to unify data from different coordinate systems, compute accurate metric distances by converting geographic coordinates to a projected system, and interoperate with external GIS tools that use specific SRIDs. Always verify the target SRID exists in `INFORMATION_SCHEMA.ST_SPATIAL_REFERENCE_SYSTEMS` before transforming.
