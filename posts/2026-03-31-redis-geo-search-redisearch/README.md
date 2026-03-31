# How to Implement Geo Search with RediSearch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Geo Search, Location

Description: Implement location-based search using RediSearch GEO fields to find nearby stores, restaurants, or users within a radius.

---

RediSearch supports GEO fields natively, allowing you to combine full-text search with geographic proximity filtering. This is useful for store locators, restaurant finders, and any application that needs to find nearby entities.

## Creating an Index with a GEO Field

Define a schema that includes a GEO field for coordinates:

```bash
FT.CREATE store_idx ON HASH PREFIX 1 store:
  SCHEMA
    name TEXT WEIGHT 5.0
    category TAG
    address TEXT
    location GEO
    rating NUMERIC SORTABLE
```

## Adding Location Data

Store each location as a Redis hash with longitude,latitude in the GEO field:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def add_store(store_id: str, name: str, category: str,
              address: str, lon: float, lat: float, rating: float):
    r.hset(f"store:{store_id}", mapping={
        "name": name,
        "category": category,
        "address": address,
        "location": f"{lon},{lat}",
        "rating": rating
    })

# Add some stores in San Francisco
add_store("sf001", "Blue Bottle Coffee", "cafe",
          "315 Linden St", -122.4259, 37.7751, 4.5)
add_store("sf002", "Tartine Bakery", "bakery",
          "600 Guerrero St", -122.4238, 37.7613, 4.7)
add_store("sf003", "Sightglass Coffee", "cafe",
          "270 7th St", -122.4098, 37.7775, 4.4)
```

## Searching by Proximity

Use the `@location:[lon lat radius unit]` filter syntax:

```python
def find_nearby(lon: float, lat: float, radius_km: float,
                category: str = None, limit: int = 10):
    geo_filter = f"@location:[{lon} {lat} {radius_km} km]"

    if category:
        query = f"(@category:{{{category}}}) {geo_filter}"
    else:
        query = geo_filter

    result = r.execute_command(
        'FT.SEARCH', 'store_idx', query,
        'SORTBY', 'rating', 'DESC',
        'LIMIT', 0, limit
    )

    count = result[0]
    docs = []
    for i in range(1, len(result), 2):
        doc_id = result[i]
        fields = dict(zip(result[i+1][0::2], result[i+1][1::2]))
        docs.append({"id": doc_id, **fields})
    return {"count": count, "results": docs}

# Find coffee shops within 2km of Union Square
nearby_cafes = find_nearby(-122.4077, 37.7879, 2, category="cafe")
```

## Combining Text and Geo Filters

Search by name and location simultaneously:

```python
def search_nearby_by_name(query: str, lon: float, lat: float,
                          radius_km: float):
    combined = f"({query}) @location:[{lon} {lat} {radius_km} km]"
    return r.execute_command(
        'FT.SEARCH', 'store_idx', combined,
        'LIMIT', 0, 10
    )

results = search_nearby_by_name("coffee", -122.4077, 37.7879, 5)
```

## Filtering by Rating and Distance

Combine GEO with numeric range filters for higher-quality results:

```python
def find_top_rated_nearby(lon: float, lat: float, radius_km: float,
                          min_rating: float = 4.0):
    query = (
        f"@location:[{lon} {lat} {radius_km} km] "
        f"@rating:[{min_rating} +inf]"
    )
    return r.execute_command(
        'FT.SEARCH', 'store_idx', query,
        'SORTBY', 'rating', 'DESC',
        'LIMIT', 0, 10
    )

top_spots = find_top_rated_nearby(-122.4077, 37.7879, 3, min_rating=4.5)
```

## Using FT.AGGREGATE for Distance Grouping

Group nearby results by category:

```bash
FT.AGGREGATE store_idx "@location:[-122.4077 37.7879 5 km]"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS total
  SORTBY 2 @total DESC
```

## Summary

RediSearch GEO fields make it simple to build proximity-based search features without a dedicated geospatial database. By combining GEO filters with text queries, tag filters, and numeric ranges, you can build a rich location-aware search API entirely within Redis. Sort results by rating or distance to surface the most relevant results.
