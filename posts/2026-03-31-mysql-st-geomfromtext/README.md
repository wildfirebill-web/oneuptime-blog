# How to Use ST_GeomFromText() in MySQL for Geospatial Queries

Author: [OneUptime](https://www.github.com/OneUptime)

Tags: MySQL, SQL, Spatial, GIS, ST_GeomFromText, Database

Description: Learn how to use ST_GeomFromText() to convert Well-Known Text strings into MySQL geometry values for storing and querying geographic data with SRID support.

---

## What Is ST_GeomFromText

`ST_GeomFromText()` is a MySQL spatial function that converts a Well-Known Text (WKT) string into a binary geometry value that can be stored in a spatial column or used directly in a spatial query. WKT is a standard text format defined by the Open Geospatial Consortium for representing geometric objects.

`ST_GeomFromText` accepts any geometry type: POINT, LINESTRING, POLYGON, MULTIPOINT, MULTILINESTRING, MULTIPOLYGON, and GEOMETRYCOLLECTION. It also accepts an optional SRID (Spatial Reference Identifier) that defines the coordinate reference system.

```mermaid
flowchart LR
    A["WKT string\n'POINT(lon lat)'"] --> B["ST_GeomFromText(wkt, srid)"]
    B --> C["Binary geometry value\nstored in column"]
    C --> D["Spatial functions:\nST_Distance, ST_Within, ST_Intersects"]
```

## Syntax

```sql
ST_GeomFromText(wkt_string)
ST_GeomFromText(wkt_string, srid)

-- Type-specific aliases (same behavior, stricter type checking)
ST_PointFromText(wkt_string [, srid])
ST_LineStringFromText(wkt_string [, srid])
ST_PolyFromText(wkt_string [, srid])
ST_MPointFromText(wkt_string [, srid])
ST_MLineFromText(wkt_string [, srid])
ST_MPolyFromText(wkt_string [, srid])
ST_GeomCollFromText(wkt_string [, srid])
```

The WKT format uses X (longitude) first, then Y (latitude).

## WKT Reference

```sql
-- POINT: single coordinate
ST_GeomFromText('POINT(-74.006 40.7128)', 4326)

-- LINESTRING: ordered sequence of points
ST_GeomFromText('LINESTRING(-74.006 40.7128, -73.985 40.758)', 4326)

-- POLYGON: closed ring (first = last point)
ST_GeomFromText('POLYGON((-74.02 40.70, -73.97 40.70, -73.97 40.73, -74.02 40.73, -74.02 40.70))', 4326)

-- MULTIPOINT: set of points
ST_GeomFromText('MULTIPOINT((-74.006 40.7128), (-73.985 40.758))', 4326)

-- MULTILINESTRING: set of line segments
ST_GeomFromText('MULTILINESTRING((-74.006 40.71, -73.99 40.72), (-73.98 40.75, -73.97 40.76))', 4326)

-- MULTIPOLYGON: set of polygons
ST_GeomFromText('MULTIPOLYGON(((-74.02 40.70, -73.97 40.70, -73.97 40.73, -74.02 40.73, -74.02 40.70)), ((-73.96 40.74, -73.94 40.74, -73.94 40.76, -73.96 40.76, -73.96 40.74)))', 4326)

-- GEOMETRYCOLLECTION: mix of types
ST_GeomFromText('GEOMETRYCOLLECTION(POINT(-74.006 40.7128), LINESTRING(-74.006 40.71, -73.99 40.72))', 4326)
```

## Examples

### Create a Table and Insert Using ST_GeomFromText

```sql
CREATE TABLE places (
    id       INT          PRIMARY KEY AUTO_INCREMENT,
    name     VARCHAR(100) NOT NULL,
    location POINT        NOT NULL SRID 4326,
    SPATIAL INDEX idx_location (location)
);

INSERT INTO places (name, location) VALUES
    ('Times Square',      ST_GeomFromText('POINT(-73.9855 40.7580)', 4326)),
    ('Empire State Bldg', ST_GeomFromText('POINT(-73.9857 40.7484)', 4326)),
    ('Central Park S',    ST_GeomFromText('POINT(-73.9730 40.7648)', 4326)),
    ('Brooklyn Bridge',   ST_GeomFromText('POINT(-73.9969 40.7061)', 4326));
```

### Use ST_GeomFromText in a WHERE Clause

```sql
SET @search_zone = ST_GeomFromText(
    'POLYGON((-74.010 40.740, -73.960 40.740, -73.960 40.770, -74.010 40.770, -74.010 40.740))',
    4326
);

SELECT name
FROM places
WHERE ST_Within(location, @search_zone);
```

```text
+--------------------+
| name               |
+--------------------+
| Times Square       |
| Empire State Bldg  |
+--------------------+
```

### Validate WKT Input

```sql
-- ST_GeomFromText raises an error if the WKT is malformed
-- Use ST_IsValid to verify geometry after creation
SELECT ST_IsValid(
    ST_GeomFromText('POLYGON((-74.02 40.70, -73.97 40.70, -73.97 40.73, -74.02 40.73, -74.02 40.70))', 4326)
) AS valid_polygon;
```

```text
+--------------+
| valid_polygon|
+--------------+
| 1            |
+--------------+
```

### SRID Behavior

```sql
-- Without SRID: geometry stored with SRID 0 (Cartesian plane)
SELECT ST_SRID(ST_GeomFromText('POINT(1 2)')) AS srid_no_arg;

-- With SRID 4326: WGS84 geographic coordinates
SELECT ST_SRID(ST_GeomFromText('POINT(-74.006 40.7128)', 4326)) AS srid_4326;

-- SRID mismatch causes error in MySQL 8.0+
-- Both geometries in a spatial function must have the same SRID
```

```text
+-------------+
| srid_no_arg |
+-------------+
| 0           |
+-------------+
+-----------+
| srid_4326 |
+-----------+
| 4326      |
+-----------+
```

### Combine ST_GeomFromText with Spatial Functions

```sql
-- Distance from Times Square to each place
SET @times_square = ST_GeomFromText('POINT(-73.9855 40.7580)', 4326);

SELECT
    name,
    ROUND(ST_Distance_Sphere(location, @times_square)) AS distance_meters
FROM places
ORDER BY distance_meters;
```

```text
+--------------------+-----------------+
| name               | distance_meters |
+--------------------+-----------------+
| Times Square       | 0               |
| Empire State Bldg  | 1069            |
| Central Park S     | 1488            |
| Brooklyn Bridge    | 6094            |
+--------------------+-----------------+
```

### Parameterized WKT in Application Code

When building WKT dynamically, always validate coordinates and use parameterized queries:

```sql
-- Good: use a prepared statement with ? for the WKT string
PREPARE stmt FROM 'INSERT INTO places (name, location) VALUES (?, ST_GeomFromText(?, 4326))';
SET @name = 'Rockefeller Center';
SET @wkt  = 'POINT(-73.9787 40.7587)';
EXECUTE stmt USING @name, @wkt;
DEALLOCATE PREPARE stmt;
```

## ST_GeomFromText vs ST_MakePoint

| Function          | Input Format           | Use Case                         |
|-------------------|------------------------|----------------------------------|
| ST_GeomFromText   | WKT string             | Any geometry type from text      |
| ST_MakePoint(x,y) | Two numeric arguments  | POINT only, cleaner for coords   |

```sql
-- Equivalent POINT creation
ST_GeomFromText('POINT(-73.9855 40.7580)', 4326)
ST_SRID(ST_MakePoint(-73.9855, 40.7580), 4326)
```

## Best Practices

- Always pass an SRID when working with real-world geographic data. Use 4326 for GPS/WGS84 coordinates.
- Use type-specific functions (`ST_PointFromText`, `ST_PolyFromText`) to get an error when the WKT type does not match what the column expects.
- Store WKT generation in the application layer and use parameterized queries to avoid SQL injection.
- Remember the WKT order is X (longitude) first, Y (latitude) second.

## Summary

`ST_GeomFromText(wkt, srid)` converts a WKT string into a MySQL binary geometry value. It supports all geometry types and accepts an optional SRID for coordinate reference system context. Use it to insert spatial data into geometry columns, build query predicates with `ST_Within` and `ST_Intersects`, and calculate distances with `ST_Distance_Sphere`. Always include SRID 4326 for geographic coordinates.
