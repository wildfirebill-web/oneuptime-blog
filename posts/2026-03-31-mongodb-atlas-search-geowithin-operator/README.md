# How to Use the geoWithin Operator in MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, GeoWithin, Geospatial, Atlas

Description: Learn how to use the Atlas Search geoWithin operator to find documents with coordinates inside a defined geographic polygon or circle.

---

The `geoWithin` operator in MongoDB Atlas Search finds documents whose GeoJSON coordinates fall inside a specified geometric boundary. It supports circles and polygons, enabling location-based filtering within Atlas Search aggregation pipelines.

## Requirements

- The field must contain valid GeoJSON coordinates
- The field must be indexed as `geo` type in the Atlas Search index mapping

## Index Configuration

Configure the search index to include a geo field:

```json
{
  "mappings": {
    "fields": {
      "location": {
        "type": "geo"
      }
    }
  }
}
```

Sample document format:

```json
{
  "name": "Coffee Shop",
  "location": {
    "type": "Point",
    "coordinates": [-73.935242, 40.730610]
  }
}
```

## Circle Search (Radius-Based)

Find all coffee shops within 2 km of a point. The radius is in meters:

```javascript
db.places.aggregate([
  {
    $search: {
      geoWithin: {
        path: "location",
        circle: {
          center: {
            type: "Point",
            coordinates: [-73.935242, 40.730610]
          },
          radius: 2000
        }
      }
    }
  },
  {
    $project: {
      name: 1,
      location: 1
    }
  }
])
```

## Polygon Search

Find stores within a defined neighborhood boundary:

```javascript
db.stores.aggregate([
  {
    $search: {
      geoWithin: {
        path: "location",
        geometry: {
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
  },
  {
    $project: {
      name: 1,
      address: 1
    }
  }
])
```

Note: The polygon's first and last coordinate must be identical to close the ring.

## MultiPolygon Search

Search across multiple geographic areas simultaneously:

```javascript
db.deliveryZones.aggregate([
  {
    $search: {
      geoWithin: {
        path: "location",
        geometry: {
          type: "MultiPolygon",
          coordinates: [
            [[[10.0, 48.0], [11.0, 48.0], [11.0, 49.0], [10.0, 49.0], [10.0, 48.0]]],
            [[[12.0, 48.0], [13.0, 48.0], [13.0, 49.0], [12.0, 49.0], [12.0, 48.0]]]
          ]
        }
      }
    }
  }
])
```

## Combining geoWithin with Text Search

Find restaurants inside a polygon that match a cuisine type:

```javascript
db.restaurants.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: "italian pasta",
              path: ["name", "description"]
            }
          }
        ],
        filter: [
          {
            geoWithin: {
              path: "location",
              circle: {
                center: {
                  type: "Point",
                  coordinates: [-73.98, 40.75]
                },
                radius: 1500
              }
            }
          }
        ]
      }
    }
  },
  {
    $project: {
      name: 1,
      cuisine: 1,
      score: { $meta: "searchScore" }
    }
  },
  { $sort: { score: -1 } }
])
```

## geoWithin vs MongoDB $geoWithin

```text
Operator             | Context       | Notes
---------------------|---------------|-----------------------------
$geoWithin (MQL)     | find/aggregate| Uses 2dsphere index
geoWithin (Atlas)    | $search stage  | Uses Atlas geo index
```

Atlas `geoWithin` integrates with Atlas Search features like compound queries and scoring, while MQL `$geoWithin` works with standard MongoDB queries.

## Summary

The `geoWithin` operator enables location-aware filtering in Atlas Search pipelines using circles or polygons. Used as a `filter` inside `compound`, it restricts results to a geographic boundary while preserving full-text relevance scoring for mixed location-and-keyword search applications.

