# How to Use polygonsSymDifference() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Geospatial, polygonsSymDifference, Polygon, Spatial Analysis, Set Operation

Description: Learn how to use polygonsSymDifference() in ClickHouse to compute the symmetric difference of two polygons - the area in either polygon but not both.

---

`polygonsSymDifference()` computes the symmetric difference of two polygons - the region that belongs to exactly one of the two input polygons, excluding their overlap. This is the spatial equivalent of the XOR set operation.

## Understanding Symmetric Difference

Given two overlapping squares A and B:
- `polygonsIntersection(A, B)` returns the overlapping region
- `polygonsUnion(A, B)` returns everything covered by A or B
- `polygonsSymDifference(A, B)` returns everything covered by A or B but NOT both

## Basic Example

```sql
SELECT polygonsSymDifference(
    [[(0.0, 0.0), (4.0, 0.0), (4.0, 4.0), (0.0, 4.0), (0.0, 0.0)]],
    [[(2.0, 2.0), (6.0, 2.0), (6.0, 6.0), (2.0, 6.0), (2.0, 2.0)]]
) AS sym_diff;
```

The result is the two "wings" of each square that do not overlap - a shape with a square hole in the center where the squares intersect.

## Use Case: Territory Change Detection

When delivery zones are redrawn, `polygonsSymDifference()` reveals which areas changed coverage:

```sql
SELECT
    zone_id,
    polygonsSymDifference(old_polygon, new_polygon) AS changed_area,
    polygonArea(polygonsSymDifference(old_polygon, new_polygon)) AS change_area_size
FROM zone_history
WHERE version_old = 3 AND version_new = 4
ORDER BY change_area_size DESC;
```

## Identifying Gaps and Overlaps in Coverage

Two adjacent zones should ideally cover their combined territory without overlap and without gaps. The symmetric difference of the union vs. the expected total area highlights problems:

```sql
SELECT
    zone_a,
    zone_b,
    polygonArea(polygonsSymDifference(z1.polygon, z2.polygon)) AS non_shared_area,
    polygonArea(polygonsIntersection(z1.polygon, z2.polygon))  AS shared_area
FROM zone_pairs AS z1
JOIN zone_pairs AS z2 USING (region_id)
WHERE z1.zone_id != z2.zone_id;
```

## Measuring Change Over Time

You can compute what fraction of a zone changed between time periods:

```sql
SELECT
    zone_id,
    polygonArea(polygonsSymDifference(prev.polygon, curr.polygon)) AS changed_area,
    polygonArea(polygonsUnion(prev.polygon, curr.polygon))         AS total_area,
    round(
        100.0 * polygonArea(polygonsSymDifference(prev.polygon, curr.polygon))
              / polygonArea(polygonsUnion(prev.polygon, curr.polygon)),
        2
    ) AS change_pct
FROM zones AS curr
JOIN zones_prev_month AS prev USING (zone_id);
```

## Polygon Format Reminder

All polygon functions use `Array(Array(Tuple(Float64, Float64)))`. The inner array holds a ring's vertices; the outer array holds all rings (exterior + holes):

```sql
-- Rectangle with no holes
[[(x1,y1), (x2,y2), (x3,y3), (x4,y4), (x1,y1)]]

-- Rectangle with a hole
[[(x1,y1), ...outer ring...],
 [(hx1,hy1), ...inner hole ring...]]
```

## Summary

`polygonsSymDifference()` returns the area exclusive to each polygon - neither the intersection nor the area outside both. It is the right tool for change detection, coverage gap analysis, and territory audit workflows. Combine it with `polygonArea()` to quantify the size of the changed regions.
