# How to Find Nearest Neighbors Using Spatial Queries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Spatial Query, Geolocation, Database, GIS

Description: Learn how to find the nearest neighbors to a point in MySQL using spatial indexes, ST_Distance_Sphere, and Haversine queries.

---

## What Is a Nearest Neighbor Query?

A nearest neighbor (NN) query finds the K closest records to a given location. This is used in applications like "show me the 5 closest restaurants," "find the nearest warehouse," or "suggest nearby users." MySQL supports this with spatial distance functions.

## Setting Up the Table

```sql
CREATE TABLE places (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  coords POINT NOT NULL SRID 4326,
  SPATIAL INDEX (coords)
);

INSERT INTO places (name, coords) VALUES
  ('Place A', ST_GeomFromText('POINT(40.7128 -74.0060)', 4326)),
  ('Place B', ST_GeomFromText('POINT(40.7580 -73.9855)', 4326)),
  ('Place C', ST_GeomFromText('POINT(40.6892 -74.0445)', 4326)),
  ('Place D', ST_GeomFromText('POINT(40.7282 -73.7949)', 4326)),
  ('Place E', ST_GeomFromText('POINT(40.7484 -73.9967)', 4326));
```

## Finding K Nearest Neighbors with ST_Distance_Sphere()

Order all results by distance from the reference point and take the top K:

```sql
SET @ref = ST_GeomFromText('POINT(40.7300 -74.0000)', 4326);

SELECT
  name,
  ROUND(ST_Distance_Sphere(coords, @ref)) AS distance_meters
FROM places
ORDER BY ST_Distance_Sphere(coords, @ref)
LIMIT 3;
```

This returns the 3 nearest records along with their distances in meters.

## Using a Bounding Box to Speed Up NN Queries

On large tables, ordering the full table by distance is slow. Add a bounding box filter to pre-select candidates:

```sql
SET @lat = 40.7300;
SET @lng = -74.0000;
SET @search_radius_m = 5000;  -- 5 km bounding box
SET @ref = ST_GeomFromText(CONCAT('POINT(', @lat, ' ', @lng, ')'), 4326);

-- Approximate degree delta for 5 km
SET @delta = 0.045;

SELECT
  name,
  ROUND(ST_Distance_Sphere(coords, @ref)) AS distance_meters
FROM places
WHERE MBRContains(
  ST_GeomFromText(CONCAT(
    'POLYGON((',
    @lat - @delta, ' ', @lng - @delta, ',',
    @lat + @delta, ' ', @lng - @delta, ',',
    @lat + @delta, ' ', @lng + @delta, ',',
    @lat - @delta, ' ', @lng + @delta, ',',
    @lat - @delta, ' ', @lng - @delta,
    '))'
  ), 4326),
  coords
)
ORDER BY distance_meters
LIMIT 5;
```

## Nearest Neighbor with Haversine (Decimal Columns)

If your table stores lat/lng as `DECIMAL` columns:

```sql
CREATE TABLE drivers (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  lat DECIMAL(10, 7),
  lng DECIMAL(10, 7),
  INDEX (lat, lng)
);

SET @lat = 40.7300;
SET @lng = -74.0000;

SELECT
  name,
  lat,
  lng,
  (6371000 * ACOS(
    COS(RADIANS(@lat)) * COS(RADIANS(lat)) *
    COS(RADIANS(lng) - RADIANS(@lng)) +
    SIN(RADIANS(@lat)) * SIN(RADIANS(lat))
  )) AS distance_meters
FROM drivers
ORDER BY distance_meters
LIMIT 5;
```

## Excluding the Reference Point Itself

When the reference point is also a row in the same table, exclude it by ID:

```sql
SET @my_id = 1;
SET @ref = (SELECT coords FROM places WHERE id = @my_id);

SELECT
  id,
  name,
  ROUND(ST_Distance_Sphere(coords, @ref)) AS distance_meters
FROM places
WHERE id != @my_id
ORDER BY distance_meters
LIMIT 5;
```

## Performance Considerations

- MySQL's spatial index (`SPATIAL INDEX`) is an R-tree index. It speeds up `MBRContains()`, `ST_Within()`, and bounding box queries but not pure `ORDER BY ST_Distance_Sphere()`.
- Always combine a bounding box pre-filter with a distance sort for large tables.
- Consider increasing `innodb_buffer_pool_size` if spatial index pages are frequently evicted.

```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

## Summary

MySQL nearest neighbor queries use `ST_Distance_Sphere()` to compute distances and `ORDER BY ... LIMIT K` to return the closest records. For large tables, pair the distance sort with a bounding box pre-filter using `MBRContains()` to leverage spatial indexes. The Haversine formula is an alternative when data is stored as decimal columns.
