# How to Build a Delivery Tracking System with Redis Geospatial

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Geospatial, Delivery, Tracking, Real-Time

Description: Build a real-time delivery tracking system with Redis Geospatial commands that updates courier positions and estimates arrival times live.

---

Delivery tracking requires storing and querying hundreds of moving courier positions per second. Redis Geospatial stores coordinates in a Sorted Set and supports fast updates and proximity queries, making it ideal for fleet tracking.

## Updating Courier Positions

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def update_courier_position(courier_id: str, lat: float, lon: float):
    now = time.time()
    pipe = r.pipeline()
    # Update geo index
    pipe.geoadd("couriers:geo", [lon, lat, courier_id])
    # Store last seen timestamp
    pipe.hset(f"courier:{courier_id}", mapping={
        "lat": lat, "lon": lon,
        "last_update": now,
    })
    pipe.execute()

def is_courier_active(courier_id: str, max_stale_seconds: int = 60) -> bool:
    last_update = r.hget(f"courier:{courier_id}", "last_update")
    if not last_update:
        return False
    return time.time() - float(last_update) < max_stale_seconds
```

## Order-to-Courier Assignment

```python
def assign_order(order_id: str, courier_id: str, destination_lat: float,
                 destination_lon: float):
    r.hset(f"order:{order_id}", mapping={
        "courier_id": courier_id,
        "destination_lat": destination_lat,
        "destination_lon": destination_lon,
        "status": "in_progress",
        "created_at": time.time(),
    })

def get_order_status(order_id: str) -> dict:
    order = r.hgetall(f"order:{order_id}")
    if not order:
        return {}

    courier_id = order.get("courier_id")
    if not courier_id:
        return {**order, "courier_position": None}

    # Get current courier position
    pos = r.geopos("couriers:geo", courier_id)
    courier_pos = pos[0] if pos and pos[0] else None

    return {**order, "courier_position": courier_pos}
```

## Distance to Delivery

```python
def get_distance_to_delivery(order_id: str) -> dict:
    order = r.hgetall(f"order:{order_id}")
    courier_id = order.get("courier_id")

    if not courier_id:
        return {"error": "no courier assigned"}

    dest_lat = float(order["destination_lat"])
    dest_lon = float(order["destination_lon"])

    # Add destination temporarily for distance calculation
    dest_key = f"order:{order_id}:dest_temp"
    r.geoadd("couriers:geo", [dest_lon, dest_lat, dest_key])
    distance_km = r.geodist("couriers:geo", courier_id, dest_key, unit="km")
    r.zrem("couriers:geo", dest_key)

    # Rough ETA: assume 30 km/h average city speed
    eta_minutes = (float(distance_km or 0) / 30) * 60

    return {
        "distance_km": round(float(distance_km or 0), 2),
        "eta_minutes": round(eta_minutes, 0),
    }
```

## Finding Couriers Near a Pickup

```python
def find_available_couriers(pickup_lat: float, pickup_lon: float,
                             radius_km: float = 5) -> list:
    results = r.geosearch(
        "couriers:geo",
        longitude=pickup_lon, latitude=pickup_lat,
        radius=radius_km, unit="km",
        sort="ASC", count=10,
        withdist=True,
    )
    available = []
    for courier_id, dist in results:
        if "order:" not in courier_id and is_courier_active(courier_id):
            available.append({"courier": courier_id,
                               "distance_km": round(float(dist), 2)})
    return available
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor the courier position update API - if updates stop arriving, ETAs become stale and customer trust erodes.

```bash
redis-cli GEOPOS couriers:geo courier_001
```

## Summary

Redis Geospatial tracks courier positions with GEOADD updates and enables proximity queries with GEOSEARCH. Storing metadata (status, timestamps) in Hashes alongside geo indexes allows rich order and courier state retrieval in single pipeline calls. ETA estimation uses geodistance combined with an average speed constant as a quick first approximation.
