# How to Use S2 Geometry Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, S2 Geometry, Geospatial, geoToS2, s2ToGeo, s2GetNeighbors, Location

Description: Learn how to use S2 geometry functions in ClickHouse including geoToS2(), s2ToGeo(), s2GetNeighbors(), and s2RectAdd() for spherical spatial indexing.

---

S2 is Google's spherical geometry library that maps the Earth's surface onto a unit sphere and subdivides it into hierarchical cells. ClickHouse includes a suite of S2 functions that allow you to encode coordinates as S2 cell IDs, find neighbours, build bounding regions, and convert back to latitude/longitude.

## Converting Coordinates to S2 Cell IDs

`geoToS2(longitude, latitude)` converts a WGS84 coordinate pair to an S2 cell ID at the finest resolution (level 30):

```sql
SELECT geoToS2(37.6156, 55.7522) AS s2_cell_id;
```

```text
s2_cell_id
-------------------
4836318965958897664
```

Note the argument order is `(longitude, latitude)`, matching most ClickHouse geo functions.

## Converting S2 IDs Back to Coordinates

`s2ToGeo(s2_id)` returns a tuple `(longitude, latitude)` for the center of the S2 cell:

```sql
SELECT
    s2_id,
    s2ToGeo(s2_id).1 AS longitude,
    s2ToGeo(s2_id).2 AS latitude
FROM locations
LIMIT 5;
```

## Finding Neighboring Cells

`s2GetNeighbors(s2_id)` returns an array of the 4 edge-adjacent S2 cells at the same level:

```sql
SELECT s2GetNeighbors(geoToS2(37.6156, 55.7522)) AS neighbors;
```

This is useful for proximity lookups - you can check whether a point's S2 cell is in the neighbor list of a target cell:

```sql
SELECT count() AS nearby_events
FROM events
WHERE has(s2GetNeighbors(geoToS2(37.6156, 55.7522)), geoToS2(longitude, latitude));
```

## Building a Bounding Rectangle

`s2RectAdd(rect, s2_id)` incrementally expands an S2 latitude/longitude rectangle to contain a given cell. Start with `s2RectUnion()` to build bounding boxes over groups of points:

```sql
SELECT
    zone_id,
    s2RectUnion(geoToS2(longitude, latitude)) AS bounding_rect
FROM delivery_zones
GROUP BY zone_id;
```

## Checking Cell Containment

`s2RectContains(rect, s2_id)` checks whether a bounding rectangle contains a given S2 cell:

```sql
SELECT
    event_id,
    s2RectContains(zone_bounding_rect, geoToS2(longitude, latitude)) AS in_zone
FROM events
CROSS JOIN (
    SELECT s2RectUnion(geoToS2(longitude, latitude)) AS zone_bounding_rect
    FROM zone_points
    WHERE zone_id = 1
) AS zone
LIMIT 10;
```

## Indexing with S2

Because S2 IDs are UInt64 values, you can create a primary or skip index on them for fast spatial range scans:

```sql
CREATE TABLE events_s2
(
    event_id    UInt64,
    s2_id       UInt64,
    event_time  DateTime,
    payload     String
)
ENGINE = MergeTree
ORDER BY (s2_id, event_time);
```

## Comparison with H3

| Feature | H3 | S2 |
|---------|----|----|
| Cell shape | Hexagonal | Quadrilateral |
| Neighbour count | 6 | 4 |
| Hierarchy levels | 16 | 30 |
| ClickHouse support | Full | Core functions |

## Summary

ClickHouse's S2 functions - `geoToS2()`, `s2ToGeo()`, `s2GetNeighbors()`, `s2RectAdd()`, and `s2RectContains()` - enable spherical spatial indexing using Google's S2 library. S2 cell IDs are UInt64 values that sort well, making them efficient primary key components for geospatial tables. Use S2 when you need quadrilateral cells or when your pipeline already produces S2 IDs.
