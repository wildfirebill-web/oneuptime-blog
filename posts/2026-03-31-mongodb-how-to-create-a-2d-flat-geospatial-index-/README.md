# How to Create a 2d Flat Geospatial Index in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Geospatial, 2d Index, Database

Description: Learn how to create a 2d flat geospatial index in MongoDB for querying coordinate pairs on a flat 2D plane, suitable for game maps and non-spherical coordinates.

---

## Overview

The `2d` index in MongoDB is designed for coordinate pairs on a flat, two-dimensional plane - as opposed to the `2dsphere` index which models the Earth's spherical surface. The `2d` index is appropriate for legacy coordinate data, game maps, custom coordinate systems, or any application where geographic curvature is not relevant.

## When to Use 2d vs 2dsphere

```text
Use 2d index when:
- Data uses legacy coordinate pairs (not GeoJSON)
- Coordinates represent a flat plane (game maps, floor plans, custom systems)
- Migrating legacy MongoDB applications

Use 2dsphere index when:
- Data represents real-world geographic locations (lat/lng)
- Accurate distance calculations on Earth's surface are needed
- Using GeoJSON format
```

## Storing Data for a 2d Index

The `2d` index uses legacy coordinate pairs - either an array `[x, y]` or an embedded document with named fields:

```javascript
// Array format
db.gameObjects.insertMany([
  { name: "Castle", loc: [120, 75] },
  { name: "Village", loc: [95, 60] },
  { name: "Forest", loc: [150, 80] }
])

// Or embedded document format
db.gameObjects.insertMany([
  { name: "Castle", loc: { x: 120, y: 75 } }
])
```

## Creating the 2d Index

```javascript
db.gameObjects.createIndex({ loc: "2d" })
```

Specify custom coordinate bounds (default is -180 to 180):

```javascript
db.mapData.createIndex(
  { loc: "2d" },
  { min: 0, max: 1000 }
)
```

## Querying with $near

Find documents near a point, sorted by distance:

```javascript
db.gameObjects.find({
  loc: {
    $near: [120, 75],
    $maxDistance: 30
  }
})
```

The `$maxDistance` for a `2d` index is in the same units as the coordinate plane.

## Querying with $geoWithin

Find documents within a rectangular box:

```javascript
db.gameObjects.find({
  loc: {
    $geoWithin: {
      $box: [
        [80, 50],   // bottom-left corner
        [160, 100]  // top-right corner
      ]
    }
  }
})
```

Within a circle:

```javascript
db.gameObjects.find({
  loc: {
    $geoWithin: {
      $center: [
        [120, 75],  // center point
        50          // radius in coordinate units
      ]
    }
  }
})
```

Within a polygon:

```javascript
db.gameObjects.find({
  loc: {
    $geoWithin: {
      $polygon: [
        [100, 60],
        [140, 60],
        [140, 90],
        [100, 90]
      ]
    }
  }
})
```

## Using $geoNear in Aggregation with a 2d Index

```javascript
db.gameObjects.aggregate([
  {
    $geoNear: {
      near: [120, 75],
      distanceField: "distance",
      maxDistance: 50,
      spherical: false
    }
  },
  { $limit: 5 }
])
```

Set `spherical: false` to use flat plane distance calculations with the `2d` index.

## Compound 2d Indexes

Combine the `2d` index with additional fields:

```javascript
db.gameObjects.createIndex({ loc: "2d", type: 1 })
```

This supports filtered proximity queries like "find enemy buildings within 50 units."

## Limitations of 2d Indexes

```text
- Cannot store GeoJSON types (Points, Polygons, etc.)
- Does not account for Earth's curvature
- Only supports coordinate pair values (not complex geometries)
- The 2d index type is considered legacy in modern applications
- Maximum coordinate range is configurable but finite
```

## Summary

The `2d` flat geospatial index in MongoDB is suited for applications that use coordinates on a two-dimensional plane rather than Earth's spherical surface. It supports proximity queries (`$near`), bounding box queries (`$box`), circle queries (`$center`), and polygon queries using legacy coordinate pair format. For modern applications dealing with real-world geographic data, prefer the `2dsphere` index with GeoJSON. Reserve the `2d` index for legacy data migrations, game coordinates, or custom 2D mapping systems.
