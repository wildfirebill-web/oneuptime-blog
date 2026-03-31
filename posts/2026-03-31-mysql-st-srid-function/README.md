# How to Use ST_SRID() in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Spatial, Geometry, Function, SRID

Description: Learn how to use MySQL's ST_SRID() function to get and set the Spatial Reference System Identifier (SRID) of a geometry value.

---

## What is an SRID?

A Spatial Reference System Identifier (SRID) is a numeric code that defines the coordinate system and projection for a geometry. SRID 0 means Cartesian (flat plane) with no real-world reference. SRID 4326 is the WGS 84 geographic coordinate system used by GPS, Google Maps, and most modern GIS data.

The SRID is embedded in every MySQL geometry value. `ST_SRID()` lets you read and change it.

## Reading the SRID of a Geometry

```sql
-- Get the SRID of a geometry
SELECT ST_SRID(ST_GeomFromText('POINT(40.7128 -74.0060)', 4326)) AS srid;
```

```text
+------+
| srid |
+------+
| 4326 |
+------+
```

```sql
-- SRID 0 is the default when none is specified
SELECT ST_SRID(ST_GeomFromText('POINT(10 20)')) AS srid;
```

```text
+------+
| srid |
+------+
|    0 |
+------+
```

## Setting a New SRID

Passing a second argument to `ST_SRID()` returns a copy of the geometry with the specified SRID:

```sql
-- Change the SRID label without transforming coordinates
SELECT ST_AsText(
  ST_SRID(ST_GeomFromText('POINT(10 20)', 0), 4326)
) AS reassigned_geom;
```

**Important:** `ST_SRID(geom, new_srid)` only changes the SRID label - it does not reproject or transform the coordinates. Use `ST_Transform()` to actually convert coordinates between coordinate systems.

## Checking Column SRIDs

Query the INFORMATION_SCHEMA to see what SRID is enforced on a column:

```sql
SELECT
  TABLE_NAME,
  COLUMN_NAME,
  SRS_ID,
  GEOMETRY_TYPE_NAME
FROM INFORMATION_SCHEMA.ST_GEOMETRY_COLUMNS
WHERE TABLE_SCHEMA = 'mydb';
```

## Enforcing SRID on a Column

Columns defined with an explicit SRID reject geometries with a different SRID:

```sql
CREATE TABLE locations (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100),
  coords POINT NOT NULL SRID 4326,
  SPATIAL INDEX (coords)
);

-- This succeeds
INSERT INTO locations (name, coords) VALUES (
  'New York',
  ST_GeomFromText('POINT(40.7128 -74.0060)', 4326)
);

-- This fails: SRID mismatch
INSERT INTO locations (name, coords) VALUES (
  'Los Angeles',
  ST_GeomFromText('POINT(34.0522 -118.2437)', 0)
);
```

## Practical Use: Validating Geometry SRIDs

Before performing spatial operations, verify all geometries share the same SRID:

```sql
SELECT
  id,
  name,
  ST_SRID(coords) AS srid
FROM locations
WHERE ST_SRID(coords) != 4326;
```

This query finds any rows with an unexpected SRID that might cause errors in spatial calculations.

## ST_SRID vs ST_Transform

| Function | What It Does |
|---|---|
| `ST_SRID(geom, new_srid)` | Relabels the SRID without changing coordinates |
| `ST_Transform(geom, new_srid)` | Converts coordinates to a different coordinate system |

Use `ST_SRID()` to fix a wrong label. Use `ST_Transform()` to actually reproject data.

## Summary

`ST_SRID()` reads the SRID of a geometry when called with one argument, and relabels it when called with two. Always enforce a consistent SRID on spatial columns to prevent mixed-SRID errors in queries. When you genuinely need to convert coordinates between systems, use `ST_Transform()` instead of `ST_SRID()`.
