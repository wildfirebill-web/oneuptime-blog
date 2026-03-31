# How to Create a 2d Flat Geospatial Index in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Geospatial, 2d Index, Flat Plane

Description: Learn how to create a 2d geospatial index in MongoDB for querying coordinates on a flat plane, suitable for game maps and non-spherical spatial data.

---

## Overview

The 2d index in MongoDB supports queries for location data stored as coordinate pairs on a flat, Euclidean plane. Unlike the 2dsphere index which models the curvature of the Earth, the 2d index treats coordinates as points on a flat surface. It is useful for game maps, floor plans, 2D layouts, and legacy applications that predate GeoJSON support.

## Creating a 2d Index

```javascript
// Create a 2d index on a location field
db.locations.createIndex({ coordinates: "2d" })
```

Documents store coordinates as a legacy coordinate pair: `[x, y]` or `{ x: number, y: number }`:

```javascript
db.locations.insertMany([
  { name: "Player Spawn", coordinates: [10, 20] },
  { name: "Treasure Chest", coordinates: [45, 80] },
  { name: "Boss Arena", coordinates: [90, 90] },
  { name: "Shop", coordinates: [25, 30] }
])
```

## Specifying Coordinate Bounds

By default, MongoDB assumes coordinates are in the range [-180, 180]. For game maps or custom grids, specify the min/max:

```javascript
// Create index for a 1000x1000 game world
db.gameEntities.createIndex(
  { position: "2d" },
  { min: 0, max: 1000 }
)
```

## $near - Find Closest Points

```javascript
// Find the 5 nearest locations to point [30, 40]
db.locations.find({
  coordinates: { $near: [30, 40] }
}).limit(5)

// With maximum distance
db.locations.find({
  coordinates: {
    $near: [30, 40],
    $maxDistance: 50
  }
}).limit(10)
```

## $geoWithin - Find Points Inside a Shape

```javascript
// $box - rectangular area
db.locations.find({
  coordinates: {
    $geoWithin: { $box: [[0, 0], [50, 60]] }
  }
})

// $circle - circular area
db.locations.find({
  coordinates: {
    $geoWithin: {
      $centerSphere: [[30, 40], 0.5]  // center, radius in radians
    }
  }
})

// $center - circular area using flat distance
db.locations.find({
  coordinates: {
    $geoWithin: {
      $center: [[30, 40], 25]  // center, radius in index units
    }
  }
})

// $polygon - polygonal area
db.locations.find({
  coordinates: {
    $geoWithin: {
      $polygon: [[0, 0], [0, 50], [50, 50], [50, 0]]
    }
  }
})
```

## Practical Example - Game Map Entity Search

```javascript
// Game map: 500x500 unit grid
db.gameMap.createIndex(
  { position: "2d" },
  { min: 0, max: 500 }
)

db.gameMap.insertMany([
  { type: "enemy", name: "Goblin", position: [100, 150], hp: 50 },
  { type: "enemy", name: "Orc", position: [200, 300], hp: 200 },
  { type: "item", name: "Health Potion", position: [110, 160] },
  { type: "npc", name: "Merchant", position: [50, 50] }
])

// Find all entities within 30 units of the player at [120, 160]
db.gameMap.find({
  position: {
    $near: [120, 160],
    $maxDistance: 30
  }
})

// Find all enemies within a combat zone
db.gameMap.find({
  type: "enemy",
  position: {
    $geoWithin: { $box: [[80, 120], [220, 320]] }
  }
})
```

## 2d vs 2dsphere Comparison

```text
| Feature              | 2d Index              | 2dsphere Index         |
|----------------------|-----------------------|------------------------|
| Coordinate model     | Flat plane            | Earth sphere           |
| Data format          | [x, y] pairs          | GeoJSON objects        |
| Distance calculation | Euclidean             | Spherical (Haversine)  |
| Use case             | Game maps, floor plans| Real-world locations   |
| Default range        | [-180, 180]           | N/A (GeoJSON)          |
| $near result order   | Euclidean distance    | Spherical distance     |
```

## Compound 2d Index

You can combine a 2d index with other fields in a compound index, but the 2d field must come first:

```javascript
db.locations.createIndex({ coordinates: "2d", category: 1 })

// Now this query can use the compound index efficiently
db.locations.find({
  coordinates: { $near: [30, 40] },
  category: "enemy"
}).limit(10)
```

## Summary

The 2d index stores and queries coordinate pairs on a flat Euclidean plane, making it suitable for game maps, floor plans, and any non-spherical spatial data. Create it with `createIndex({ field: "2d" })` and optionally set `min`/`max` bounds. Use `$near` for proximity searches sorted by distance, and `$geoWithin` with `$box`, `$center`, or `$polygon` to find points within shapes. For real-world geographic data, use 2dsphere indexes with GeoJSON instead.
