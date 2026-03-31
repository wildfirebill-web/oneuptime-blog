# How to Use polygonsWithin() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Geospatial, Polygon, polygonsWithin, Geo Function

Description: Learn how to use polygonsWithin() in ClickHouse to check if one polygon is fully contained within another for geospatial containment queries.

---

The `polygonsWithin()` function in ClickHouse tests whether one polygon is completely contained within another. It returns 1 if the first polygon is fully within the second polygon, and 0 otherwise.

## Function Signature

```sql
polygonsWithin(inner_polygon, outer_polygon)
```

Both arguments are polygons represented as arrays of rings: `Array(Array(Tuple(Float64, Float64)))`. The first ring in each array is the outer boundary; additional rings represent holes.

## Basic Example

```sql
SELECT polygonsWithin(
    [[(2., 2.), (8., 2.), (8., 8.), (2., 8.), (2., 2.)]],
    [[(0., 0.), (10., 0.), (10., 10.), (0., 10.), (0., 0.)]]
) AS is_within
```

```text
is_within
---------
1
```

The smaller square (2-8 on each axis) is fully within the larger square (0-10), so the result is 1.

## Filtering Records by Containment

A common use case is checking whether a smaller zone lies entirely within a larger service area:

```sql
SELECT
    zone_name,
    polygonsWithin(zone_boundary, service_area) AS is_covered
FROM delivery_zones
WHERE polygonsWithin(zone_boundary, service_area) = 1
```

This filters delivery zones that are fully covered by the overall service area - useful for billing and SLA verification.

## Hierarchical Region Validation

```sql
SELECT
    sub.region_name AS sub_region,
    parent.region_name AS parent_region,
    polygonsWithin(sub.boundary, parent.boundary) AS valid_hierarchy
FROM regions sub
INNER JOIN regions parent ON parent.level = sub.level - 1
    AND sub.parent_id = parent.id
ORDER BY valid_hierarchy ASC
```

This validates that all sub-regions are properly contained within their parent regions - useful for data quality checks on geographic data.

## Combining with pointInPolygon

You can layer `polygonsWithin()` with `pointInPolygon()` to build more complex spatial queries:

```sql
SELECT
    user_id,
    latitude,
    longitude
FROM user_events
WHERE
    pointInPolygon((longitude, latitude), large_area) = 1
    AND NOT pointInPolygon((longitude, latitude), restricted_zone)
```

## Checking Coverage of Multiple Sub-zones

```sql
SELECT
    district_id,
    countIf(polygonsWithin(block_boundary, district_boundary) = 1) AS covered_blocks,
    count() AS total_blocks
FROM city_blocks cb
JOIN districts d ON cb.district_id = d.id
GROUP BY district_id
```

This query summarizes how many blocks within each district are fully contained in the district boundary - a data quality metric.

## Performance Tips

- Pre-simplify polygons to reduce vertex counts before storing them in ClickHouse.
- Use `Array(Array(Tuple(Float64, Float64)))` columns and define appropriate `ORDER BY` keys to enable efficient filtering.
- For large datasets with repeated containment checks against the same parent polygon, consider storing the result in a materialized column.

## Summary

`polygonsWithin()` in ClickHouse is a straightforward but powerful function for polygon containment testing. It is ideal for validating geographic hierarchies, filtering locations by coverage area, and ensuring data consistency in geospatial datasets stored in ClickHouse.
