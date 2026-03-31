# How to Build a Ride-Sharing Matching System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Geospatial, Ride-Sharing, Matching, Real-Time

Description: Build a ride-sharing driver-to-passenger matching system with Redis that finds the nearest available driver in milliseconds.

---

Ride-sharing matching is a real-time geospatial problem: given a passenger's location, find the nearest available driver and assign the trip. Redis Geo commands with driver availability tracking make this possible at scale.

## Driver Availability Management

```python
import redis
import time
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

DRIVERS_GEO_KEY = "drivers:available:geo"

def driver_go_online(driver_id: str, lat: float, lon: float):
    pipe = r.pipeline()
    pipe.geoadd(DRIVERS_GEO_KEY, [lon, lat, driver_id])
    pipe.hset(f"driver:{driver_id}", mapping={
        "status": "available",
        "lat": lat, "lon": lon,
        "last_ping": time.time(),
    })
    pipe.execute()

def driver_go_offline(driver_id: str):
    pipe = r.pipeline()
    pipe.zrem(DRIVERS_GEO_KEY, driver_id)
    pipe.hset(f"driver:{driver_id}", "status", "offline")
    pipe.execute()

def update_driver_position(driver_id: str, lat: float, lon: float):
    status = r.hget(f"driver:{driver_id}", "status")
    if status == "available":
        r.geoadd(DRIVERS_GEO_KEY, [lon, lat, driver_id])
    pipe = r.pipeline()
    pipe.hset(f"driver:{driver_id}", mapping={
        "lat": lat, "lon": lon, "last_ping": time.time()
    })
    pipe.execute()
```

## Finding and Matching a Driver

```python
def find_nearest_driver(passenger_lat: float, passenger_lon: float,
                        search_radius_km: float = 10) -> dict:
    results = r.geosearch(
        DRIVERS_GEO_KEY,
        longitude=passenger_lon, latitude=passenger_lat,
        radius=search_radius_km, unit="km",
        sort="ASC", count=1,
        withdist=True,
    )

    if not results:
        return {"driver": None, "message": "No drivers available"}

    driver_id, dist = results[0]
    driver_info = r.hgetall(f"driver:{driver_id}")

    return {
        "driver_id": driver_id,
        "distance_km": round(float(dist), 2),
        "eta_minutes": round(float(dist) / 30 * 60, 0),
        "driver_info": driver_info,
    }

def assign_trip(trip_id: str, driver_id: str, passenger_id: str,
                origin_lat: float, origin_lon: float,
                dest_lat: float, dest_lon: float):
    # Mark driver as busy
    pipe = r.pipeline()
    pipe.zrem(DRIVERS_GEO_KEY, driver_id)
    pipe.hset(f"driver:{driver_id}", "status", "on_trip")

    # Create trip record
    pipe.hset(f"trip:{trip_id}", mapping={
        "driver_id": driver_id,
        "passenger_id": passenger_id,
        "origin_lat": origin_lat, "origin_lon": origin_lon,
        "dest_lat": dest_lat, "dest_lon": dest_lon,
        "status": "in_progress",
        "started_at": time.time(),
    })
    pipe.execute()
```

## Completing a Trip

```python
def complete_trip(trip_id: str, final_lat: float, final_lon: float):
    trip = r.hgetall(f"trip:{trip_id}")
    driver_id = trip.get("driver_id")

    if driver_id:
        # Return driver to available pool
        driver_go_online(driver_id, final_lat, final_lon)
        r.hset(f"trip:{trip_id}", "status", "completed")
```

## Passenger Request Queue

```python
def queue_passenger_request(passenger_id: str, lat: float, lon: float):
    request = json.dumps({
        "passenger_id": passenger_id,
        "lat": lat, "lon": lon,
        "timestamp": time.time()
    })
    r.lpush("requests:pending", request)
    r.expire("requests:pending", 3600)
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your trip-assignment service and alert if matching latency exceeds 500ms - riders expect near-instant driver assignment.

```bash
redis-cli ZCARD drivers:available:geo
```

## Summary

Redis Geo tracks available driver positions and GEOSEARCH finds the nearest available driver in O(N + log M) time. Removing a driver from the available geo index on assignment prevents double-booking. Trip lifecycle management uses Hashes to store origin, destination, and status, with drivers returning to the geo pool on trip completion.
