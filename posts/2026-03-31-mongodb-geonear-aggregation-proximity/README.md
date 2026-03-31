# How to Use $geoNear to Find Documents by Proximity in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Geospatial, $geoNear, Pipeline

Description: Learn how to use the $geoNear aggregation stage in MongoDB to find and sort documents by proximity to a given point with distance metadata.

---

The `$geoNear` aggregation stage combines a geospatial search with a sort and distance calculation in a single pipeline stage. Unlike the `$near` query operator, `$geoNear` must be the first stage in a pipeline and outputs a configurable distance field alongside each result document.

## Basic Setup

Your collection needs a geospatial index before `$geoNear` will work.

```js
db.places.createIndex({ location: "2dsphere" });
```

Insert some sample documents:

```js
db.places.insertMany([
  { name: "Coffee Shop", location: { type: "Point", coordinates: [-73.9857, 40.7484] } },
  { name: "Library",     location: { type: "Point", coordinates: [-73.9810, 40.7530] } },
  { name: "Park",        location: { type: "Point", coordinates: [-73.9920, 40.7450] } }
]);
```

## Running $geoNear

Find all places within 1 km of a reference point, sorted by distance:

```js
db.places.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [-73.9857, 40.7484] },
      distanceField: "distanceMeters",
      maxDistance: 1000,
      spherical: true
    }
  }
]);
```

Each result includes a `distanceMeters` field showing how far the document is from the query point.

## Adding a Filter

Use the `query` option inside `$geoNear` to filter results before the distance calculation:

```js
db.places.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [-73.9857, 40.7484] },
      distanceField: "distanceMeters",
      maxDistance: 2000,
      query: { category: "food" },
      spherical: true
    }
  }
]);
```

## Converting Units

By default, distances are in meters when using a `2dsphere` index. Use `distanceMultiplier` to convert to kilometers or miles:

```js
db.places.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [-73.9857, 40.7484] },
      distanceField: "distanceKm",
      distanceMultiplier: 0.001,
      spherical: true
    }
  },
  { $project: { name: 1, distanceKm: { $round: ["$distanceKm", 2] } } }
]);
```

## Chaining Downstream Stages

After `$geoNear`, you can chain any other aggregation stages. Here, results are limited to the nearest 5 places and projected cleanly:

```js
db.places.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [-73.9857, 40.7484] },
      distanceField: "distanceMeters",
      spherical: true
    }
  },
  { $limit: 5 },
  { $project: { _id: 0, name: 1, distanceMeters: { $round: ["$distanceMeters", 0] } } }
]);
```

## Key Options Reference

| Option | Description |
|--------|-------------|
| `near` | The reference GeoJSON point or legacy coordinate pair |
| `distanceField` | Field name to store the calculated distance |
| `maxDistance` | Maximum distance in meters (2dsphere) or radians (2d) |
| `minDistance` | Minimum distance to exclude very close results |
| `query` | Additional match filter applied to documents |
| `spherical` | Set `true` for 2dsphere indexes |
| `distanceMultiplier` | Multiply distance by this factor before storing |

## Summary

`$geoNear` is the recommended aggregation stage for proximity searches in MongoDB because it combines filtering, distance calculation, and sorting in one efficient step. Always place it first in the pipeline, add a `2dsphere` index on your location field, and use `distanceMultiplier` when you need distances in units other than meters.
