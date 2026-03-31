# How to Store Spatial Data in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Spatial Data, GIS, Geometry

Description: Learn how to store and query spatial data in MySQL using geometry data types, WKT format, and spatial indexes for geographic applications.

---

## MySQL Spatial Data Types

MySQL supports the OGC (Open Geospatial Consortium) geometry data types:

| Type | Description |
|---|---|
| `POINT` | Single location (x, y) |
| `LINESTRING` | Sequence of connected points |
| `POLYGON` | Closed shape with an area |
| `MULTIPOINT` | Collection of points |
| `MULTILINESTRING` | Collection of line strings |
| `MULTIPOLYGON` | Collection of polygons |
| `GEOMETRYCOLLECTION` | Mixed collection of any geometry types |
| `GEOMETRY` | Generic type that can hold any geometry |

All types are stored using the SRID (Spatial Reference ID) to define the coordinate system.

## Creating a Table with Spatial Columns

```sql
CREATE TABLE locations (
  id          INT          NOT NULL AUTO_INCREMENT,
  name        VARCHAR(200) NOT NULL,
  description TEXT,
  coordinates POINT        NOT NULL SRID 4326,  -- WGS84 (GPS coordinates)
  boundary    POLYGON,
  PRIMARY KEY (id),
  SPATIAL INDEX idx_coordinates (coordinates)
);
```

`SRID 4326` is the WGS84 coordinate system used by GPS (latitude/longitude).

## Inserting Spatial Data

Use Well-Known Text (WKT) format with `ST_GeomFromText()`:

```sql
-- Insert a point (longitude, latitude order for SRID 4326)
INSERT INTO locations (name, coordinates) VALUES
  ('Eiffel Tower',
   ST_GeomFromText('POINT(2.2945 48.8584)', 4326)),
  ('Big Ben',
   ST_GeomFromText('POINT(-0.1246 51.5007)', 4326)),
  ('Tokyo Tower',
   ST_GeomFromText('POINT(139.7454 35.6586)', 4326));
```

Insert a polygon (district boundary):

```sql
INSERT INTO locations (name, boundary) VALUES
  ('Central Park',
   ST_GeomFromText('POLYGON((
     -73.9812 40.7680,
     -73.9583 40.7680,
     -73.9583 40.7640,
     -73.9812 40.7640,
     -73.9812 40.7680
   ))', 4326));
```

## Querying Spatial Data

Retrieve coordinates as WKT:

```sql
SELECT name, ST_AsText(coordinates) AS wkt
FROM locations;
```

```text
+--------------+-----------------------------+
| name         | wkt                         |
+--------------+-----------------------------+
| Eiffel Tower | POINT(2.2945 48.8584)        |
| Big Ben      | POINT(-0.1246 51.5007)       |
+--------------+-----------------------------+
```

Retrieve as GeoJSON:

```sql
SELECT name, ST_AsGeoJSON(coordinates) AS geojson
FROM locations;
```

```text
{"type": "Point", "coordinates": [2.2945, 48.8584]}
```

## Spatial Index for Performance

A SPATIAL INDEX accelerates spatial queries:

```sql
-- Add a spatial index on an existing column
ALTER TABLE locations ADD SPATIAL INDEX idx_coord (coordinates);
```

Spatial indexes only work on `NOT NULL` columns with a defined SRID.

## Using Coordinates from Your Application

```python
import mysql.connector

conn = mysql.connector.connect(host='localhost', user='root', password='secret', database='geo')
cursor = conn.cursor()

lat, lng = 48.8584, 2.2945

cursor.execute("""
    INSERT INTO locations (name, coordinates)
    VALUES (%s, ST_GeomFromText(%s, 4326))
""", ('My Location', f'POINT({lng} {lat})'))

conn.commit()
```

## Selecting X and Y Coordinates

```sql
SELECT
  name,
  ST_X(coordinates) AS longitude,
  ST_Y(coordinates) AS latitude
FROM locations;
```

## Checking SRID

```sql
SELECT name, ST_SRID(coordinates) AS srid
FROM locations;
```

## Summary

MySQL stores spatial data using geometry types like `POINT`, `LINESTRING`, and `POLYGON`. Use `ST_GeomFromText('POINT(lng lat)', 4326)` to insert WGS84 coordinates, and add a `SPATIAL INDEX` for efficient geographic queries. Retrieve data with `ST_AsText()` for WKT or `ST_AsGeoJSON()` for GeoJSON format.
