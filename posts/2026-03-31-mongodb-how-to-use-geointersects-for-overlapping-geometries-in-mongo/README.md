# How to Use $geoIntersects for Overlapping Geometries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Geospatial, $geoIntersects, Geometry, Spatial Analysis

Description: Use MongoDB's $geoIntersects operator to find documents whose geometry overlaps with a specified shape, enabling point-in-polygon and region-overlap queries.

---

## What Is $geoIntersects?

`$geoIntersects` finds documents where the stored geometry field intersects with (overlaps or touches) a specified GeoJSON geometry. It works with all GeoJSON types: Point, LineString, Polygon, MultiPolygon, etc.

Key differences from `$geoWithin`:
- `$geoWithin`: The stored geometry must be ENTIRELY within the specified shape
- `$geoIntersects`: The stored geometry must INTERSECT (overlap, touch, or be contained by) the specified shape

## Common Use Cases

- Point-in-polygon (is this coordinate inside this region?)
- Overlapping service territories
- Route/road intersections
- Finding regions that overlap a search area
- Coverage map queries

## Basic $geoIntersects Query

```javascript
// Create 2dsphere index
db.regions.createIndex({ geometry: "2dsphere" })

// Find all regions that intersect a search polygon
db.regions.find({
  geometry: {
    $geoIntersects: {
      $geometry: {
        type: "Polygon",
        coordinates: [[
          [-74.05, 40.68],
          [-73.90, 40.68],
          [-73.90, 40.88],
          [-74.05, 40.88],
          [-74.05, 40.68]
        ]]
      }
    }
  }
})
```

## Point-in-Polygon: Is a Point Inside Any Region?

```javascript
// Store delivery zones as polygons
db.deliveryZones.insertMany([
  {
    name: "Downtown",
    geometry: {
      type: "Polygon",
      coordinates: [[
        [-74.0200, 40.7000],
        [-73.9700, 40.7000],
        [-73.9700, 40.7500],
        [-74.0200, 40.7500],
        [-74.0200, 40.7000]
      ]]
    },
    deliveryCost: 2.99
  },
  {
    name: "Uptown",
    geometry: {
      type: "Polygon",
      coordinates: [[
        [-74.0200, 40.7500],
        [-73.9300, 40.7500],
        [-73.9300, 40.8500],
        [-74.0200, 40.8500],
        [-74.0200, 40.7500]
      ]]
    },
    deliveryCost: 4.99
  }
])

db.deliveryZones.createIndex({ geometry: "2dsphere" })

// Find which zone a customer is in
const customerLocation = {
  type: "Point",
  coordinates: [-73.9857, 40.7484]
};

const zone = await db.collection("deliveryZones").findOne({
  geometry: {
    $geoIntersects: { $geometry: customerLocation }
  }
});

console.log(zone ? `Customer is in ${zone.name} - cost: $${zone.deliveryCost}` : "No delivery zone");
```

## Check if Two Polygons Overlap

```javascript
// Service territories for two companies
db.territories.insertMany([
  {
    company: "Company A",
    territory: {
      type: "Polygon",
      coordinates: [[
        [-74.05, 40.68], [-73.95, 40.68],
        [-73.95, 40.78], [-74.05, 40.78], [-74.05, 40.68]
      ]]
    }
  },
  {
    company: "Company B",
    territory: {
      type: "Polygon",
      coordinates: [[
        [-74.00, 40.73], [-73.90, 40.73],
        [-73.90, 40.83], [-74.00, 40.83], [-74.00, 40.73]
      ]]
    }
  }
])

// Find all territories that overlap with Company A's territory
const companyA = await db.collection("territories").findOne({ company: "Company A" });

const overlapping = await db.collection("territories").find({
  company: { $ne: "Company A" },
  territory: {
    $geoIntersects: { $geometry: companyA.territory }
  }
}).toArray();

console.log(`${overlapping.length} territories overlap with Company A`);
```

## $geoIntersects with LineString

Find roads or routes that cross a given area:

```javascript
// Store routes as LineStrings
db.routes.insertMany([
  {
    name: "Route 66",
    path: {
      type: "LineString",
      coordinates: [
        [-118.2437, 34.0522],
        [-117.2898, 34.1083],
        [-116.5453, 33.8303]
      ]
    }
  }
])

db.routes.createIndex({ path: "2dsphere" })

// Find routes that pass through a search polygon
db.routes.find({
  path: {
    $geoIntersects: {
      $geometry: {
        type: "Polygon",
        coordinates: [[
          [-118.50, 33.90], [-116.00, 33.90],
          [-116.00, 34.20], [-118.50, 34.20], [-118.50, 33.90]
        ]]
      }
    }
  }
})
```

## $geoIntersects vs $geoWithin

```javascript
// $geoWithin: the stored Point must be INSIDE the polygon
db.restaurants.find({
  location: { $geoWithin: { $geometry: cityBoundary } }
})

// $geoIntersects: the stored Point can be AT or INSIDE the polygon
// For Points: $geoIntersects and $geoWithin return the same result
db.restaurants.find({
  location: { $geoIntersects: { $geometry: cityBoundary } }
})

// For Polygons: $geoIntersects also finds polygons that PARTIALLY overlap
// $geoWithin only finds polygons ENTIRELY within the search area
```

## Find All Points in Multiple Regions

```javascript
// Check if a point intersects any of multiple service areas
const customerPoint = { type: "Point", coordinates: [-73.9857, 40.7484] };

const serviceAreas = await db.collection("serviceAreas").find({
  boundary: {
    $geoIntersects: { $geometry: customerPoint }
  }
}).toArray();

if (serviceAreas.length > 0) {
  console.log("Served by:", serviceAreas.map(a => a.name).join(", "));
} else {
  console.log("Not in any service area");
}
```

## Node.js Utility Function

```javascript
async function findIntersectingZones(longitude, latitude, collectionName, geometryField) {
  const client = new MongoClient(process.env.MONGODB_URI);
  await client.connect();

  const point = { type: "Point", coordinates: [longitude, latitude] };

  const zones = await client.db("myapp").collection(collectionName).find({
    [geometryField]: {
      $geoIntersects: { $geometry: point }
    }
  }).toArray();

  await client.close();
  return zones;
}

// Find all zones containing a customer
const zones = await findIntersectingZones(-73.9857, 40.7484, "serviceZones", "boundary");
```

## Summary

`$geoIntersects` finds documents whose geometry field intersects with a given GeoJSON shape, including points inside polygons, overlapping regions, and routes crossing an area. For point-in-polygon queries, `$geoIntersects` and `$geoWithin` produce equivalent results; for polygon vs. polygon, `$geoIntersects` finds partial overlaps while `$geoWithin` requires complete containment. Create a `2dsphere` index on the geometry field for query performance and use it for geofencing, service territory analysis, and spatial overlap detection.
