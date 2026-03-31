# How to Track Fleet Vehicles in Real-Time with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Fleet, Geospatial, Vehicle Tracking, Real-Time

Description: Track a fleet of vehicles in real-time with Redis Geospatial, storing live positions, speed, and route deviation alerts.

---

Fleet tracking systems ingest GPS pings from hundreds or thousands of vehicles per second. Redis handles this write throughput natively and supports real-time geospatial queries for dispatch, routing, and compliance monitoring.

## Ingesting Vehicle Positions

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def update_vehicle(vehicle_id: str, lat: float, lon: float,
                   speed_kmh: float, heading: float):
    now = time.time()
    pipe = r.pipeline()
    # Live geo position
    pipe.geoadd("fleet:geo", [lon, lat, vehicle_id])
    # Vehicle telemetry
    pipe.hset(f"vehicle:{vehicle_id}", mapping={
        "lat": lat, "lon": lon,
        "speed_kmh": speed_kmh,
        "heading": heading,
        "last_update": now,
        "status": "moving" if speed_kmh > 2 else "stopped",
    })
    # Breadcrumb trail (last 100 positions)
    pipe.lpush(f"vehicle:{vehicle_id}:trail", f"{lat},{lon},{now}")
    pipe.ltrim(f"vehicle:{vehicle_id}:trail", 0, 99)
    pipe.execute()
```

## Getting Fleet Overview

```python
def get_all_vehicles() -> list:
    # Get all vehicle IDs from the geo index
    vehicle_ids = [v for v in r.zrange("fleet:geo", 0, -1)]
    if not vehicle_ids:
        return []

    pipe = r.pipeline()
    for vid in vehicle_ids:
        pipe.hgetall(f"vehicle:{vid}")
    telemetry_list = pipe.execute()

    return [
        {"vehicle_id": vid, **tel}
        for vid, tel in zip(vehicle_ids, telemetry_list)
        if tel
    ]

def get_vehicles_in_area(lat: float, lon: float, radius_km: float) -> list:
    results = r.geosearch(
        "fleet:geo",
        longitude=lon, latitude=lat,
        radius=radius_km, unit="km",
        sort="ASC", withdist=True,
    )
    return [
        {"vehicle_id": vid, "distance_km": round(float(dist), 2)}
        for vid, dist in results
    ]
```

## Stale Vehicle Detection

```python
def get_stale_vehicles(max_age_seconds: int = 120) -> list:
    now = time.time()
    vehicles = get_all_vehicles()
    return [
        v for v in vehicles
        if now - float(v.get("last_update", 0)) > max_age_seconds
    ]
```

## Speeding Alert

```python
SPEED_LIMITS = {
    "highway": 120,
    "urban": 60,
    "residential": 40,
}

def check_speed_violation(vehicle_id: str, zone_type: str = "urban") -> bool:
    speed = float(r.hget(f"vehicle:{vehicle_id}", "speed_kmh") or 0)
    limit = SPEED_LIMITS.get(zone_type, 60)
    if speed > limit:
        r.sadd(f"fleet:violations:{zone_type}", vehicle_id)
        return True
    return False
```

## Vehicle Breadcrumb Trail

```python
def get_vehicle_trail(vehicle_id: str, last_n: int = 20) -> list:
    raw = r.lrange(f"vehicle:{vehicle_id}:trail", 0, last_n - 1)
    trail = []
    for entry in raw:
        parts = entry.split(",")
        trail.append({
            "lat": float(parts[0]),
            "lon": float(parts[1]),
            "timestamp": float(parts[2]),
        })
    return trail
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your GPS ingestion API - tracking pipeline failures means vehicles disappear from the map silently.

```bash
redis-cli ZCARD fleet:geo
redis-cli HGETALL vehicle:truck_001
```

## Summary

Redis Geo + Hash pairs track every vehicle's position and telemetry state with a single pipeline write per GPS ping. GEOSEARCH enables zone-based queries for dispatch assignment. Breadcrumb trails stored in trimmed Lists provide last-known route history without unbounded memory growth.
