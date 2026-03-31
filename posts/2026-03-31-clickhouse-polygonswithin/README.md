# How to Use polygonsWithin() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Geospatial, polygonsWithin, Polygon, Containment, Spatial Query

Description: Learn how to use polygonsWithin() in ClickHouse to test whether one polygon is completely contained within another for geofencing and zone analysis.

---

`polygonsWithin(polygon1, polygon2)` returns 1 if `polygon1` is completely contained within `polygon2` and 0 otherwise. It is the polygon-level counterpart to `pointInPolygon()` and is useful for hierarchical zone containment checks.

## Basic Syntax

```sql
SELECT polygonsWithin(
    [[(1.0, 1.0), (3.0, 1.0), (3.0, 3.0), (1.0, 3.0), (1.0, 1.0)]],
    [[(0.0, 0.0), (5.0, 0.0), (5.0, 5.0), (0.0, 5.0), (0.0, 0.0)]]
) AS is_within;
```

```text
is_within
---------
1
```

The smaller 2x2 square is completely inside the larger 5x5 square, so the result is 1.

## Checking That a Delivery Zone is Within a City Boundary

```sql
SELECT
    dz.zone_id,
    dz.zone_name,
    polygonsWithin(dz.polygon, city.boundary) AS within_city
FROM delivery_zones AS dz
CROSS JOIN (SELECT boundary FROM city_boundaries WHERE city_name = 'London') AS city;
```

## Finding All Sub-Zones Within a Parent Region

```sql
SELECT zone_id, zone_name
FROM service_zones
WHERE polygonsWithin(
    polygon,
    (SELECT polygon FROM regions WHERE region_name = 'Greater London')
) = 1;
```

## Validating Zone Hierarchy

When you have a hierarchy of nested zones (country -> region -> district), validate that each child is within its parent:

```sql
SELECT
    child.zone_id,
    child.zone_name,
    parent.zone_name AS parent_name,
    polygonsWithin(child.polygon, parent.polygon) AS valid_hierarchy
FROM zones AS child
JOIN zones AS parent ON child.parent_zone_id = parent.zone_id
WHERE valid_hierarchy = 0;
```

Any rows returned indicate hierarchy violations where a child zone extends outside its parent.

## Difference Between polygonsWithin() and pointInPolygon()

- `pointInPolygon()` tests a single point against a polygon
- `polygonsWithin()` tests full polygon containment - every point in `polygon1` must be inside `polygon2`

```sql
-- Test if a single point is in a polygon
SELECT pointInPolygon((2.5, 2.5), [(0.0,0.0),(5.0,0.0),(5.0,5.0),(0.0,5.0),(0.0,0.0)]) AS in_poly;

-- Test if one polygon is fully within another
SELECT polygonsWithin(
    [[(2.0,2.0),(3.0,2.0),(3.0,3.0),(2.0,3.0),(2.0,2.0)]],
    [[(0.0,0.0),(5.0,0.0),(5.0,5.0),(0.0,5.0),(0.0,0.0)]]
) AS poly_within;
```

## Performance Tip

For large zone tables, pre-filter using bounding box comparisons before calling `polygonsWithin()`:

```sql
SELECT zone_id
FROM micro_zones
WHERE min_lat >= (SELECT min_lat FROM macro_zones WHERE id = 1)
  AND max_lat <= (SELECT max_lat FROM macro_zones WHERE id = 1)
  AND polygonsWithin(polygon, (SELECT polygon FROM macro_zones WHERE id = 1)) = 1;
```

## Summary

`polygonsWithin()` returns 1 when the first polygon is completely enclosed by the second polygon. It is the right function for hierarchy validation, zone containment checks, and finding which delivery zones fall entirely inside a city boundary. Pre-filter with bounding-box comparisons to keep performance fast on large tables.
