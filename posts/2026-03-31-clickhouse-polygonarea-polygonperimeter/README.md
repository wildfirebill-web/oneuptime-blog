# How to Use polygonArea() and polygonPerimeter() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Geospatial, polygonArea, polygonPerimeter, Polygon, Spatial Measurement

Description: Learn how to compute the area and perimeter of polygons in ClickHouse using polygonArea() and polygonPerimeter() for spatial measurement and zone analysis.

---

ClickHouse provides `polygonArea()` and `polygonPerimeter()` for computing the area and perimeter of polygon geometries represented as arrays of coordinate rings. These functions use planar (Euclidean) geometry, so units depend on the coordinate system you use.

## Basic Syntax

Both functions accept a polygon in `Array(Array(Tuple(Float64, Float64)))` format:

```sql
SELECT
    polygonArea([[(0.0, 0.0), (4.0, 0.0), (4.0, 3.0), (0.0, 3.0), (0.0, 0.0)]])   AS area,
    polygonPerimeter([[(0.0, 0.0), (4.0, 0.0), (4.0, 3.0), (0.0, 3.0), (0.0, 0.0)]]) AS perimeter;
```

```text
area | perimeter
-----+----------
12   | 14
```

The 4x3 rectangle has area 12 and perimeter 14, as expected.

## Working with Geographic Coordinates

When using longitude/latitude in degrees, `polygonArea()` returns a value in squared degrees. For an approximate conversion to square kilometers near the equator, multiply by roughly 12,321 (111 km/degree squared):

```sql
SELECT
    zone_id,
    zone_name,
    polygonArea(polygon)                      AS area_sq_deg,
    round(polygonArea(polygon) * 12321.0, 2) AS area_sq_km_approx
FROM delivery_zones
ORDER BY area_sq_km_approx DESC;
```

For accurate spherical area, project coordinates to a metric system first or use H3 cell counts as a proxy.

## Computing Perimeter for Service Zone Analysis

```sql
SELECT
    zone_id,
    zone_name,
    round(polygonPerimeter(polygon) * 111.0, 2) AS perimeter_km_approx
FROM delivery_zones
ORDER BY perimeter_km_approx DESC;
```

## Ranking Zones by Compactness

The isoperimetric ratio (area / perimeter^2) measures how compact a shape is - a circle maximizes this value:

```sql
SELECT
    zone_id,
    zone_name,
    polygonArea(polygon)        AS area,
    polygonPerimeter(polygon)   AS perimeter,
    round(
        polygonArea(polygon) / (polygonPerimeter(polygon) * polygonPerimeter(polygon)),
        4
    ) AS compactness_ratio
FROM delivery_zones
ORDER BY compactness_ratio DESC;
```

## Polygons with Holes

Both functions correctly account for holes (inner rings). A donut shape subtracts the hole area:

```sql
SELECT
    polygonArea([
        [(0.0, 0.0), (10.0, 0.0), (10.0, 10.0), (0.0, 10.0), (0.0, 0.0)],
        [(3.0, 3.0), (7.0, 3.0), (7.0, 7.0), (3.0, 7.0), (3.0, 3.0)]
    ]) AS donut_area;
```

```text
donut_area
----------
84
```

The outer 10x10 square (area 100) minus the inner 4x4 square (area 16) gives 84.

## Finding Zones Larger Than a Threshold

```sql
SELECT zone_id, zone_name, polygonArea(polygon) AS area
FROM service_zones
WHERE polygonArea(polygon) > 0.01
ORDER BY area DESC;
```

## Summary

`polygonArea()` computes the enclosed area of a polygon and correctly handles holes in multi-ring polygons. `polygonPerimeter()` returns the total boundary length. Both use planar geometry so apply a degree-to-kilometer conversion factor when working with geographic coordinates. These functions are useful for ranking zones by size, filtering by minimum coverage area, and computing compactness metrics.
