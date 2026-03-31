# How to Use the Geo Data Types in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Types, Geospatial, Point, Ring, Polygon

Description: Learn how ClickHouse represents geospatial data using Point, Ring, Polygon, and MultiPolygon types and how to query them with geo functions.

---

ClickHouse provides a set of geospatial data types built on top of its existing type system. `Point` is a `Tuple(Float64, Float64)` representing (longitude, latitude). `Ring` is an `Array(Point)` forming a closed polygon boundary. `Polygon` is an `Array(Ring)` where the first ring is the outer boundary and subsequent rings are holes. `MultiPolygon` is an `Array(Polygon)` for multi-part regions. These types pair with geo functions like `pointInPolygon` to enable spatial queries without a separate GIS extension.

## Point: Storing Coordinates

`Point` is a type alias for `Tuple(Float64, Float64)` with convention (longitude, latitude). Enable experimental geo types with the setting if required by your ClickHouse version.

```sql
-- Enable geo types if needed (ClickHouse 22.4+)
SET allow_experimental_geo_types = 1;

CREATE TABLE locations (
    location_id UInt32,
    name        String,
    coordinates Point,
    category    LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY location_id;

INSERT INTO locations VALUES
    (1, 'Eiffel Tower',      (2.2945, 48.8584),   'landmark'),
    (2, 'Louvre Museum',     (2.3376, 48.8606),   'museum'),
    (3, 'Notre-Dame',        (2.3499, 48.8530),   'landmark'),
    (4, 'Sacre-Coeur',       (2.3431, 48.8867),   'landmark');

-- Access longitude and latitude
SELECT name, coordinates.1 AS longitude, coordinates.2 AS latitude
FROM locations;
```

## Ring: Defining a Polygon Boundary

A `Ring` is an `Array(Point)` that forms a closed loop. The first and last point do not need to be identical in ClickHouse - the ring is implicitly closed.

```sql
-- Define a simple rectangular Ring (bounding box around central Paris)
SELECT [
    (2.2700, 48.8400),
    (2.4000, 48.8400),
    (2.4000, 48.8800),
    (2.2700, 48.8800),
    (2.2700, 48.8400)
] AS paris_ring;
```

## Polygon: Outer Boundary and Holes

A `Polygon` is an `Array(Ring)`. The first element is the outer boundary; any additional elements define holes (exclusion zones).

```sql
SET allow_experimental_geo_types = 1;

CREATE TABLE delivery_zones (
    zone_id  UInt32,
    name     String,
    boundary Polygon
) ENGINE = Memory;

INSERT INTO delivery_zones VALUES
(
    1,
    'Central Paris',
    [
        -- Outer ring (simplified bounding polygon)
        [
            (2.2700, 48.8400),
            (2.4000, 48.8400),
            (2.4000, 48.8800),
            (2.2700, 48.8800),
            (2.2700, 48.8400)
        ]
        -- Additional rings here would be holes
    ]
),
(
    2,
    'Left Bank',
    [
        [
            (2.2900, 48.8400),
            (2.3700, 48.8400),
            (2.3700, 48.8550),
            (2.2900, 48.8550),
            (2.2900, 48.8400)
        ]
    ]
);
```

## pointInPolygon: Spatial Queries

Use `pointInPolygon((lon, lat), polygon)` to test whether a coordinate lies inside a polygon.

```sql
-- Find which delivery zone each location falls in
SELECT
    l.name AS location_name,
    d.name AS zone_name
FROM locations AS l
CROSS JOIN delivery_zones AS d
WHERE pointInPolygon(l.coordinates, d.boundary);

-- Inline polygon test without a table
SELECT pointInPolygon(
    (2.3376, 48.8606),  -- Louvre Museum
    [
        [
            (2.2700, 48.8400),
            (2.4000, 48.8400),
            (2.4000, 48.8800),
            (2.2700, 48.8800),
            (2.2700, 48.8400)
        ]
    ]
) AS is_inside;
```

## MultiPolygon: Multi-Part Regions

`MultiPolygon` is `Array(Polygon)` and represents a region composed of multiple disjoint polygon parts.

```sql
SET allow_experimental_geo_types = 1;

CREATE TABLE regions (
    region_id   UInt32,
    name        String,
    area        MultiPolygon
) ENGINE = Memory;

-- A MultiPolygon with two separate polygon parts
INSERT INTO regions VALUES
(
    1,
    'Two Islands',
    [
        -- First polygon (island 1)
        [[(1.0, 1.0), (2.0, 1.0), (2.0, 2.0), (1.0, 2.0), (1.0, 1.0)]],
        -- Second polygon (island 2)
        [[(4.0, 4.0), (5.0, 4.0), (5.0, 5.0), (4.0, 5.0), (4.0, 4.0)]]
    ]
);
```

## Distance Calculations

Use `greatCircleDistance` to compute the great-circle distance between two points in metres.

```sql
-- Distance in metres between Eiffel Tower and Louvre
SELECT
    a.name AS from_point,
    b.name AS to_point,
    round(greatCircleDistance(
        a.coordinates.1, a.coordinates.2,
        b.coordinates.1, b.coordinates.2
    )) AS distance_m
FROM locations AS a
CROSS JOIN locations AS b
WHERE a.location_id < b.location_id
ORDER BY distance_m ASC;
```

## Summary

ClickHouse's geo types - `Point`, `Ring`, `Polygon`, and `MultiPolygon` - provide a lightweight, SQL-native approach to geospatial storage and querying. `pointInPolygon` enables zone membership checks, and `greatCircleDistance` computes accurate geodesic distances. For simple use cases these types are sufficient; for complex GIS workloads with topology operations, index-accelerated bounding-box lookups, or projection support, a dedicated spatial database may be more appropriate.
