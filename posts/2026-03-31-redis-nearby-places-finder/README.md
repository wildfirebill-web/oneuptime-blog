# How to Build a Nearby Places Finder with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Geospatial, Nearby, Location, GEORADIUS

Description: Build a nearby places finder with Redis Geospatial commands that returns locations within a radius in milliseconds.

---

Finding nearby places - restaurants, stores, drivers - is a core feature in location-based apps. Redis Geospatial commands index coordinates in a Sorted Set and support radius queries in O(N + log M) time.

## Adding Places

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

GEO_KEY = "places:geo"

def add_place(place_id: str, latitude: float, longitude: float, name: str):
    pipe = r.pipeline()
    # Add to geo index
    pipe.geoadd(GEO_KEY, [longitude, latitude, place_id])
    # Store metadata
    pipe.hset(f"place:{place_id}", mapping={
        "name": name,
        "lat": latitude,
        "lon": longitude,
        "id": place_id,
    })
    pipe.execute()

def bulk_add_places(places: list):
    """places = [{"id": "...", "lat": ..., "lon": ..., "name": "..."}]"""
    geo_args = []
    pipe = r.pipeline()
    for p in places:
        geo_args.extend([p["lon"], p["lat"], p["id"]])
        pipe.hset(f"place:{p['id']}", mapping={
            "name": p["name"], "lat": p["lat"], "lon": p["lon"]
        })
    pipe.geoadd(GEO_KEY, geo_args)
    pipe.execute()
```

## Finding Nearby Places

```python
def find_nearby(lat: float, lon: float, radius_km: float,
                max_results: int = 20) -> list:
    results = r.geosearch(
        GEO_KEY,
        longitude=lon,
        latitude=lat,
        radius=radius_km,
        unit="km",
        sort="ASC",
        count=max_results,
        withcoord=True,
        withdist=True,
    )

    places = []
    for place_id, dist, coords in results:
        meta = r.hgetall(f"place:{place_id}")
        places.append({
            "id": place_id,
            "name": meta.get("name", ""),
            "distance_km": round(float(dist), 2),
            "lat": coords[1],
            "lon": coords[0],
        })
    return places
```

## Distance Between Two Places

```python
def distance_between(place_a: str, place_b: str, unit: str = "km") -> float:
    return r.geodist(GEO_KEY, place_a, place_b, unit=unit)
```

## Filtering by Category

Maintain category-specific geo indexes for filtered searches:

```python
def add_place_with_category(place_id: str, lat: float, lon: float,
                             category: str):
    pipe = r.pipeline()
    pipe.geoadd(f"places:geo:{category}", [lon, lat, place_id])
    pipe.geoadd(GEO_KEY, [lon, lat, place_id])
    pipe.execute()

def find_nearby_category(lat: float, lon: float, radius_km: float,
                          category: str, max_results: int = 20) -> list:
    results = r.geosearch(
        f"places:geo:{category}",
        longitude=lon, latitude=lat,
        radius=radius_km, unit="km",
        sort="ASC", count=max_results,
        withdist=True,
    )
    return [{"id": pid, "distance_km": round(float(dist), 2)}
            for pid, dist in results]
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your location API uptime and latency - geo queries should return within 10ms; SLA alerts catch performance regressions.

```bash
redis-cli GEOSEARCH places:geo FROMLONLAT -73.9857 40.7484 BYRADIUS 2 km ASC COUNT 5 WITHCOORD WITHDIST
```

## Summary

Redis GEOSEARCH returns places within a radius sorted by distance with a single command. Separate per-category geo keys enable filtered queries without client-side post-processing. Storing metadata in Hashes alongside geo keys avoids extra lookups by pipelining metadata fetches with geo queries.
