# How to Use polygonsIntersection() and polygonsUnion() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Geospatial, polygonsIntersection, polygonsUnion, Polygon, Spatial Analysis

Description: Learn how to compute polygon intersections and unions in ClickHouse using polygonsIntersection() and polygonsUnion() for geofencing and territory analysis.

---

ClickHouse provides polygon set-operations through `polygonsIntersection()` and `polygonsUnion()`. These functions operate on arrays of rings (each ring is an array of `(x, y)` tuples) and return the resulting polygon geometry, enabling geofencing overlap analysis and territory merging directly in SQL.

## Polygon Representation

ClickHouse represents polygons as `Array(Array(Tuple(Float64, Float64)))`. The outer array holds rings - the first ring is the exterior boundary and any additional rings are holes:

```sql
-- A simple square polygon centered near London
SELECT [(51.5, -0.1), (51.5, 0.1), (51.4, 0.1), (51.4, -0.1), (51.5, -0.1)] AS square;
```

For multi-ring polygons (with holes), wrap each ring in an outer array.

## polygonsIntersection()

`polygonsIntersection(polygon1, polygon2)` returns the geometric intersection of two polygons:

```sql
SELECT polygonsIntersection(
    [[(0.0, 0.0), (4.0, 0.0), (4.0, 4.0), (0.0, 4.0), (0.0, 0.0)]],
    [[(2.0, 2.0), (6.0, 2.0), (6.0, 6.0), (2.0, 6.0), (2.0, 2.0)]]
) AS intersection;
```

The result is the 2x2 square from (2,2) to (4,4) where the two squares overlap.

## Finding Overlap Between Delivery Zones

```sql
SELECT
    z1.zone_name AS zone_a,
    z2.zone_name AS zone_b,
    length(polygonsIntersection(z1.polygon, z2.polygon)[1]) > 0 AS overlaps
FROM delivery_zones AS z1
CROSS JOIN delivery_zones AS z2
WHERE z1.zone_id < z2.zone_id;
```

## polygonsUnion()

`polygonsUnion(polygon1, polygon2)` returns the polygon covering the area of either or both input polygons:

```sql
SELECT polygonsUnion(
    [[(0.0, 0.0), (4.0, 0.0), (4.0, 4.0), (0.0, 4.0), (0.0, 0.0)]],
    [[(2.0, 2.0), (6.0, 2.0), (6.0, 6.0), (2.0, 6.0), (2.0, 2.0)]]
) AS union_polygon;
```

## Merging Service Areas

When two branches merge, you may want to compute the combined service area:

```sql
SELECT polygonsUnion(
    branch_a.service_area,
    branch_b.service_area
) AS combined_area
FROM branches AS branch_a
JOIN branches AS branch_b ON branch_a.region = branch_b.region
WHERE branch_a.id = 1 AND branch_b.id = 2;
```

## Checking for Non-Empty Intersection

A fast existence check uses `polygonsIntersection()` combined with `length()`:

```sql
SELECT
    campaign_id,
    countIf(length(polygonsIntersection(campaign.area, user.home_area)[1]) > 0)
        AS users_in_area
FROM campaigns AS campaign
CROSS JOIN user_profiles AS user
GROUP BY campaign_id;
```

## Performance Considerations

Polygon operations are CPU-intensive. Reduce the candidate set before calling these functions:

```sql
SELECT polygonsIntersection(p1, p2)
FROM zones
WHERE pointInPolygon((center_lat, center_lon), bounding_box)
  AND zone_id IN (1, 2, 3);
```

Pre-filter using `pointInPolygon()` or bounding-box checks before running the full intersection computation.

## Summary

`polygonsIntersection()` and `polygonsUnion()` bring standard computational geometry operations into ClickHouse SQL. Use `polygonsIntersection()` to find overlapping zones and `polygonsUnion()` to merge territories. Both accept the same `Array(Array(Tuple(Float64, Float64)))` polygon format and return the same type, making them composable for multi-step spatial workflows.
