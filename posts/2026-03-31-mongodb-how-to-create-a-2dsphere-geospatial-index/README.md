# How to Create a 2dsphere Geospatial Index in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Geospatial, 2dsphere, Database

Description: Learn how to create a 2dsphere geospatial index in MongoDB for querying locations on a spherical Earth model using GeoJSON and proximity operators.

---

## Overview

The `2dsphere` index in MongoDB supports queries on geographic data modeled as points, lines, and polygons on a spherical surface (the Earth). It uses GeoJSON format for location data and supports proximity queries (`$near`, `$nearSphere`), range queries (`$geoWithin`), and intersection queries (`$geoIntersects`). It is the recommended geospatial index for location-based applications.

## Storing Location Data as GeoJSON

Before creating the index, ensure your documents store location data in GeoJSON format:

```javascript
db.places.insertMany([
  {
    name: "Central Park",
    location: {
      type: "Point",
      coordinates: [-73.9654, 40.7829]  // [longitude, latitude]
    }
  },
  {
    name: "Times Square",
    location: {
      type: "Point",
      coordinates: [-73.9857, 40.7580]
    }
  }
])
```

Note: GeoJSON uses `[longitude, latitude]` order, which is the opposite of the common "lat, lng" convention.

## Creating the 2dsphere Index

```javascript
db.places.createIndex({ location: "2dsphere" })
```

## Querying Nearby Locations with $near

Find places within a certain distance (in meters) from a point:

```javascript
db.places.find({
  location: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [-73.9857, 40.7580]
      },
      $maxDistance: 1000,
      $minDistance: 0
    }
  }
})
```

Results are automatically sorted by distance (nearest first).

## Querying Within a Circle with $geoWithin

Find all places within a circular area:

```javascript
db.places.find({
  location: {
    $geoWithin: {
      $centerSphere: [
        [-73.9857, 40.7580],
        1 / 6378.1  // radius in radians (1 km)
      ]
    }
  }
})
```

Or use a GeoJSON polygon to define the area:

```javascript
db.places.find({
  location: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [[
          [-74.0, 40.7],
          [-73.9, 40.7],
          [-73.9, 40.8],
          [-74.0, 40.8],
          [-74.0, 40.7]
        ]]
      }
    }
  }
})
```

## Using $geoIntersects for Shape Queries

Find documents whose geometry intersects a given GeoJSON shape:

```javascript
db.routes.find({
  path: {
    $geoIntersects: {
      $geometry: {
        type: "Polygon",
        coordinates: [[
          [-74.0, 40.7],
          [-73.9, 40.7],
          [-73.9, 40.8],
          [-74.0, 40.8],
          [-74.0, 40.7]
        ]]
      }
    }
  }
})
```

## Calculating Distance with $geoNear in Aggregation

Use `$geoNear` as the first stage of an aggregation pipeline for richer proximity queries:

```javascript
db.restaurants.aggregate([
  {
    $geoNear: {
      near: {
        type: "Point",
        coordinates: [-73.9857, 40.7580]
      },
      distanceField: "distanceMeters",
      maxDistance: 2000,
      query: { cuisine: "Italian" },
      spherical: true
    }
  },
  { $limit: 10 },
  {
    $project: {
      name: 1,
      cuisine: 1,
      distanceKm: { $divide: ["$distanceMeters", 1000] }
    }
  }
])
```

## Indexing Multiple Geometry Types

The `2dsphere` index can be created on a field that stores any GeoJSON geometry type (Point, LineString, Polygon, MultiPoint, etc.):

```javascript
// Documents can mix geometry types
db.geoData.createIndex({ geom: "2dsphere" })
```

## Compound 2dsphere Indexes

Combine a 2dsphere index with other fields for filtered proximity queries:

```javascript
db.stores.createIndex({ location: "2dsphere", category: 1, rating: -1 })
```

This supports queries like "find open pizza restaurants within 2 km sorted by rating."

## Summary

The `2dsphere` index is MongoDB's primary geospatial index for Earth-like spherical calculations. It supports GeoJSON Point, LineString, and Polygon types and enables proximity, within-region, and intersection queries. For applications requiring location search, the `$near`, `$geoWithin`, and `$geoIntersects` operators paired with a `2dsphere` index provide efficient, accurate results. Use `$geoNear` in aggregation pipelines when you need distance as a computed field alongside other filters and projections.
