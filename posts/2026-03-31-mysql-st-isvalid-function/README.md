# How to Use ST_IsValid() in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Spatial, Geometry, Function, GIS

Description: Learn how to use MySQL's ST_IsValid() function to check geometry validity and how to fix invalid geometries with ST_MakeValid().

---

## What is ST_IsValid()?

`ST_IsValid()` returns 1 if a geometry is valid according to the OGC standard, or 0 if it is not. Invalid geometries violate structural rules - for example, a polygon with self-intersecting edges, a ring that is not closed, or a multipolygon with overlapping components.

Operating on invalid geometries produces undefined results or errors in spatial functions like `ST_Area()`, `ST_Intersection()`, and `ST_Distance()`. Always validate imported geometry data.

## Basic Syntax

```sql
ST_IsValid(geometry)
```

Returns 1 (valid) or 0 (invalid). Returns NULL if the argument is NULL.

## Checking a Valid Geometry

```sql
-- A simple valid square
SELECT ST_IsValid(
  ST_GeomFromText('POLYGON((0 0, 4 0, 4 4, 0 4, 0 0))', 0)
) AS is_valid;
```

```text
+----------+
| is_valid |
+----------+
|        1 |
+----------+
```

## Detecting an Invalid Geometry

A self-intersecting (bowtie) polygon is invalid:

```sql
-- Figure-8 / bowtie polygon (self-intersecting)
SELECT ST_IsValid(
  ST_GeomFromText('POLYGON((0 0, 4 4, 4 0, 0 4, 0 0))', 0)
) AS is_valid;
```

```text
+----------+
| is_valid |
+----------+
|        0 |
+----------+
```

## Auditing a Table for Invalid Geometries

```sql
CREATE TABLE territories (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100),
  boundary POLYGON NOT NULL SRID 0,
  SPATIAL INDEX (boundary)
);

-- Find all invalid geometries
SELECT id, name
FROM territories
WHERE ST_IsValid(boundary) = 0;
```

## Fixing Invalid Geometries with ST_MakeValid()

MySQL 8.0.24 introduced `ST_MakeValid()` to repair invalid geometries:

```sql
-- Repair invalid geometries in-place
UPDATE territories
SET boundary = ST_MakeValid(boundary)
WHERE ST_IsValid(boundary) = 0;
```

`ST_MakeValid()` applies minimal changes to make the geometry valid. Self-intersecting polygons may be split into multipolygons, and collapsed rings may be removed.

## Validating Before Insert

Use `ST_IsValid()` in application code or a trigger to prevent invalid data from entering the database:

```sql
DELIMITER //
CREATE TRIGGER validate_geometry
BEFORE INSERT ON territories
FOR EACH ROW
BEGIN
  IF ST_IsValid(NEW.boundary) = 0 THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'Invalid geometry detected';
  END IF;
END//
DELIMITER ;
```

## Common Causes of Invalid Geometries

- Self-intersecting polygon rings
- Polygons with duplicate consecutive points
- Rings that are not closed (first and last point differ)
- Multipolygons with overlapping component polygons
- Polygons with fewer than 4 points (including closing point)

## Testing Edge Cases

```sql
-- Unclosed ring (first != last point) - invalid
SELECT ST_IsValid(
  ST_GeomFromText('POLYGON((0 0, 4 0, 4 4, 0 4))', 0)
) AS valid;

-- Too few points - invalid
SELECT ST_IsValid(
  ST_GeomFromText('POLYGON((0 0, 1 1, 0 0))', 0)
) AS valid;
```

## Summary

`ST_IsValid()` checks whether a geometry conforms to OGC validity rules. Use it to audit imported spatial data, prevent invalid geometries from entering your database via triggers, and identify candidates for repair with `ST_MakeValid()`. Always validate externally-sourced geometry data before running spatial analysis functions on it.
