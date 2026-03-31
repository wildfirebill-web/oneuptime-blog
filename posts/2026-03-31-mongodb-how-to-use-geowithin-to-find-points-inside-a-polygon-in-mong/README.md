# How to Use $geoWithin to Find Points Inside a Polygon in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Geospatial, $geoWithin, Polygon, Geofencing

Description: Use MongoDB's $geoWithin operator to find all documents whose location falls inside a specified polygon or other geometry for geofencing and area-based queries.

---

## What Is $geoWithin?

`$geoWithin` returns documents where the specified location field falls within a given geometry (polygon, multi-polygon, or circle). Unlike `$near`, it does NOT sort results by distance, and it does NOT require a geospatial index (though an index improves performance).

Use cases:
- Geofencing (is this device inside a zone?)
- Delivery zone assignment
- Finding all customers in a city/region
- Regional data analysis

## $geoWithin with GeoJSON Polygon

```javascript
// Create 2dsphere index for performance
db.properties.createIndex({ location: "2dsphere" })

// Find all properties inside a polygon
db.properties.find({
  location: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [[
          [-74.0479, 40.6829],   // Southwest corner
          [-73.9067, 40.6829],   // Southeast corner
          [-73.9067, 40.8820],   // Northeast corner
          [-74.0479, 40.8820],   // Northwest corner
          [-74.0479, 40.6829]    // Close the ring (same as first point)
        ]]
      }
    }
  }
})
```

Polygon rings must be closed (last coordinate = first coordinate) and follow the right-hand rule (exterior ring counterclockwise, holes clockwise).

## Insert Sample Data

```javascript
db.properties.insertMany([
  { name: "Brooklyn Loft", location: { type: "Point", coordinates: [-73.9442, 40.6782] } },
  { name: "Midtown Office", location: { type: "Point", coordinates: [-73.9857, 40.7549] } },
  { name: "Harlem Studio", location: { type: "Point", coordinates: [-73.9445, 40.8116] } },
  { name: "Queens House", location: { type: "Point", coordinates: [-73.8370, 40.7282] } }
])
```

## $geoWithin with $box (Legacy)

For rectangular queries with a `2d` index (flat geometry):

```javascript
// Bottom-left and top-right corners define the box
db.stores.find({
  location: {
    $geoWithin: {
      $box: [
        [-74.05, 40.68],   // Bottom-left [lng, lat]
        [-73.90, 40.88]    // Top-right [lng, lat]
      ]
    }
  }
})
```

Note: `$box` uses flat geometry. For accurate real-world queries, use `$geometry` with a Polygon.

## $geoWithin with $center (Flat Circle)

```javascript
// Circle using flat geometry (2d index)
// $center: [[centerX, centerY], radius] where radius is in degrees
db.places.find({
  location: {
    $geoWithin: {
      $center: [[-73.9857, 40.7484], 0.05]  // 0.05 degrees radius (~5km)
    }
  }
})
```

## $geoWithin with $centerSphere (Spherical Circle)

For accurate circular area queries on a sphere:

```javascript
// $centerSphere: [[lng, lat], radiusInRadians]
// radiusInRadians = distanceInKm / 6371

const radiusKm = 5;
const radiusRadians = radiusKm / 6371;

db.restaurants.find({
  location: {
    $geoWithin: {
      $centerSphere: [[-73.9857, 40.7484], radiusRadians]
    }
  }
})
```

`$centerSphere` works with both `2dsphere` and `2d` indexes and uses spherical geometry.

## Geofencing: Check If a Point Is Inside a Zone

```javascript
// Delivery zones collection
db.deliveryZones.insertOne({
  name: "Manhattan",
  area: {
    type: "Polygon",
    coordinates: [[
      [-74.0204, 40.6950],
      [-73.9067, 40.6950],
      [-73.9067, 40.8820],
      [-74.0204, 40.8820],
      [-74.0204, 40.6950]
    ]]
  }
})

db.deliveryZones.createIndex({ area: "2dsphere" })

// Check which zone contains a delivery address
const deliveryPoint = { type: "Point", coordinates: [-73.9857, 40.7484] };

const zone = await db.collection("deliveryZones").findOne({
  area: {
    $geoIntersects: {
      $geometry: deliveryPoint
    }
  }
});

console.log(zone ? `Delivers to ${zone.name}` : "No delivery available");
```

## Complex Multi-Polygon Query

```javascript
// Find properties inside any of multiple neighborhoods
db.properties.find({
  location: {
    $geoWithin: {
      $geometry: {
        type: "MultiPolygon",
        coordinates: [
          // Brooklyn
          [[[-74.0479, 40.5755], [-73.8334, 40.5755], [-73.8334, 40.7395], [-74.0479, 40.7395], [-74.0479, 40.5755]]],
          // Queens
          [[[-73.9629, 40.5431], [-73.7004, 40.5431], [-73.7004, 40.8007], [-73.9629, 40.8007], [-73.9629, 40.5431]]]
        ]
      }
    }
  }
})
```

## Node.js Geofencing Example

```javascript
async function isInDeliveryZone(lng, lat, zoneName) {
  const client = new MongoClient(process.env.MONGODB_URI);
  await client.connect();

  const zones = client.db('myapp').collection('deliveryZones');

  const zone = await zones.findOne({
    name: zoneName,
    area: {
      $geoIntersects: {
        $geometry: { type: "Point", coordinates: [lng, lat] }
      }
    }
  });

  await client.close();
  return zone !== null;
}

const inZone = await isInDeliveryZone(-73.9857, 40.7484, "Manhattan");
console.log(inZone ? "In delivery zone" : "Outside delivery zone");
```

## $geoWithin vs $near

| Feature | `$geoWithin` | `$near` |
|---|---|---|
| Sorts by distance | No | Yes |
| Returns distance | No | No (use $geoNear for distance) |
| Index required | No (but recommended) | Yes |
| Query shape | Polygon, box, circle | Point with radius |
| Use case | Area containment | Proximity search |

## Summary

`$geoWithin` finds documents whose location falls within a specified geometry. Use `$geometry` with a GeoJSON Polygon or MultiPolygon for accurate spherical containment queries, `$centerSphere` for circular area queries in radians, or `$box` for simple rectangular flat-geometry queries. Unlike `$near`, results are not sorted by distance. Create a `2dsphere` index for performance. Use it for geofencing, delivery zone assignment, and regional data queries where you need all points inside an area rather than nearby points.
