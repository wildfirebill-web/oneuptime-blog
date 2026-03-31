# How to Index for Geospatial Queries with Filters in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Geospatial, Index, Filter, 2dsphere

Description: Build compound geospatial indexes in MongoDB to efficiently query locations within a radius combined with category or status equality filters.

---

## Geospatial Index Types

MongoDB provides two geospatial index types:
- `2dsphere`: For spherical (Earth-like) geometry, GeoJSON coordinates. Use for real-world location data.
- `2d`: For planar geometry on a flat grid. Use for gaming maps or 2D simulations.

This guide focuses on `2dsphere`, which is the correct choice for location-based applications.

## Store Locations as GeoJSON

MongoDB geospatial operators require location data in GeoJSON format:

```javascript
db.restaurants.insertMany([
  {
    name: "The Grid Cafe",
    category: "cafe",
    isOpen: true,
    location: {
      type: "Point",
      coordinates: [-73.9857, 40.7484], // [longitude, latitude]
    },
    rating: 4.5,
  },
  {
    name: "Pizza Palace",
    category: "restaurant",
    isOpen: true,
    location: {
      type: "Point",
      coordinates: [-73.9750, 40.7612],
    },
    rating: 4.1,
  },
]);
```

## Create a 2dsphere Index

```javascript
// Standalone geospatial index
db.restaurants.createIndex({ location: "2dsphere" });
```

## Compound Geospatial Index with Filters

To efficiently filter by both location and equality fields, create a compound index. The `2dsphere` field can appear anywhere in the compound index:

```javascript
// Filter by category + location proximity
db.restaurants.createIndex({ category: 1, location: "2dsphere" });

// Filter by status + location
db.restaurants.createIndex({ isOpen: 1, location: "2dsphere" });
```

## Query Within a Radius

Use `$near` with `$maxDistance` (in meters) for proximity searches:

```javascript
const userLocation = [-73.9800, 40.7550]; // [longitude, latitude]
const radiusMeters = 1000; // 1 km

// Find open cafes within 1 km, sorted by distance (nearest first)
db.restaurants.find({
  category: "cafe",
  isOpen: true,
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: userLocation },
      $maxDistance: radiusMeters,
    },
  },
});
```

## Query Within a Polygon

Use `$geoWithin` to find locations inside an arbitrary polygon (useful for geofencing):

```javascript
db.restaurants.find({
  isOpen: true,
  location: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [
          [
            [-74.0, 40.7],
            [-73.9, 40.7],
            [-73.9, 40.8],
            [-74.0, 40.8],
            [-74.0, 40.7], // Close the ring
          ],
        ],
      },
    },
  },
});
```

`$geoWithin` does not require a geospatial index but benefits from one for performance.

## Add Distance to Results with Aggregation

Use `$geoNear` in an aggregation pipeline to include the computed distance:

```javascript
db.restaurants.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [-73.9800, 40.7550] },
      distanceField: "distanceMeters",
      maxDistance: 2000,
      query: { isOpen: true, category: "restaurant" },
      spherical: true,
    },
  },
  { $sort: { distanceMeters: 1 } },
  { $limit: 10 },
]);
```

## Validate Index Usage

```javascript
db.restaurants.find({
  category: "cafe",
  location: { $near: { $geometry: { type: "Point", coordinates: [-73.98, 40.75] }, $maxDistance: 500 } }
}).explain("executionStats");
```

Look for `GEO_NEAR_2DSPHERE` in the winning plan, confirming the geospatial index is used.

## Summary

Use `2dsphere` indexes for real-world location data stored as GeoJSON Points. Create compound indexes combining the `2dsphere` field with equality filter fields (category, status) to avoid post-index filtering. Use `$near` for proximity-sorted results, `$geoWithin` for polygon containment, and the `$geoNear` aggregation stage to include computed distances in results. Always store coordinates as `[longitude, latitude]` in GeoJSON order.
