# How to Calculate Driving Distance Estimates with Redis Geo

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Geospatial, Distance, Routing, Estimation

Description: Use Redis Geo commands to calculate straight-line and road-corrected distance estimates between locations for ETA and cost calculations.

---

Driving distance estimation powers ETAs, delivery costs, and service area checks. Redis GEODIST calculates precise Haversine (straight-line) distances in one command. A correction factor converts these to realistic road distances.

## Basic Distance Calculation

```python
import redis
import math

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def calculate_distance(lat1: float, lon1: float,
                       lat2: float, lon2: float, unit: str = "km") -> float:
    # Add both points to a temp key for GEODIST
    temp_key = "geo:distance_calc_temp"
    pipe = r.pipeline()
    pipe.geoadd(temp_key, [lon1, lat1, "point_a"])
    pipe.geoadd(temp_key, [lon2, lat2, "point_b"])
    pipe.geodist(temp_key, "point_a", "point_b", unit=unit)
    pipe.delete(temp_key)
    results = pipe.execute()
    return float(results[2] or 0)
```

## Storing Locations for Repeated Distance Queries

```python
def add_location(location_id: str, lat: float, lon: float, name: str = ""):
    pipe = r.pipeline()
    pipe.geoadd("locations:geo", [lon, lat, location_id])
    pipe.hset(f"location:{location_id}", mapping={
        "name": name, "lat": lat, "lon": lon
    })
    pipe.execute()

def distance_between_locations(loc_a: str, loc_b: str,
                                unit: str = "km") -> float:
    return float(r.geodist("locations:geo", loc_a, loc_b, unit=unit) or 0)
```

## Road Distance Correction Factors

Straight-line to road distance correction by area type:

```python
ROAD_CORRECTION_FACTORS = {
    "urban": 1.35,      # City streets are indirect
    "suburban": 1.25,
    "rural": 1.15,      # Roads more direct in rural areas
    "highway": 1.08,    # Nearly straight
}

def estimate_road_distance(straight_km: float,
                            area_type: str = "urban") -> float:
    factor = ROAD_CORRECTION_FACTORS.get(area_type, 1.3)
    return round(straight_km * factor, 2)

def estimate_eta(straight_km: float, area_type: str = "urban") -> dict:
    road_km = estimate_road_distance(straight_km, area_type)
    speeds = {"urban": 30, "suburban": 50, "rural": 70, "highway": 100}
    speed = speeds.get(area_type, 40)
    eta_hours = road_km / speed
    return {
        "straight_line_km": round(straight_km, 2),
        "road_km": road_km,
        "eta_minutes": round(eta_hours * 60, 0),
        "area_type": area_type,
    }
```

## Multi-Stop Distance Chain

```python
def calculate_route_distance(waypoints: list, unit: str = "km") -> dict:
    """waypoints = [{"id": "...", "lat": ..., "lon": ...}]"""
    if len(waypoints) < 2:
        return {"total_km": 0, "legs": []}

    temp_key = "geo:route_temp"
    pipe = r.pipeline()
    for wp in waypoints:
        pipe.geoadd(temp_key, [wp["lon"], wp["lat"], wp["id"]])
    pipe.execute()

    legs = []
    total = 0.0
    for i in range(len(waypoints) - 1):
        dist = float(r.geodist(temp_key, waypoints[i]["id"],
                               waypoints[i + 1]["id"], unit=unit) or 0)
        legs.append({
            "from": waypoints[i]["id"],
            "to": waypoints[i + 1]["id"],
            "distance_km": round(dist, 2),
        })
        total += dist

    r.delete(temp_key)
    return {"total_km": round(total, 2), "legs": legs}
```

## Service Area Check

```python
def is_within_service_area(lat: float, lon: float,
                             depot_id: str, max_km: float = 50) -> bool:
    temp_key = "geo:service_check"
    user_id = "user_check_point"
    pipe = r.pipeline()
    pipe.geoadd(temp_key, [lon, lat, user_id])
    # Copy depot from locations
    depot_pos = r.geopos("locations:geo", depot_id)
    if not depot_pos or not depot_pos[0]:
        return False
    pipe.geoadd(temp_key, [depot_pos[0][0], depot_pos[0][1], "depot"])
    pipe.geodist(temp_key, user_id, "depot", unit="km")
    pipe.delete(temp_key)
    results = pipe.execute()
    dist = float(results[2] or 0)
    return dist <= max_km
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor your distance calculation API - it is often called on every order placement, so latency spikes directly impact checkout conversion.

```bash
redis-cli GEODIST locations:geo warehouse_nyc warehouse_la km
```

## Summary

Redis GEODIST calculates Haversine distances in O(log N) time once locations are stored in a geo key. A road correction factor (1.15-1.35x) converts straight-line distances to realistic road estimates. For multi-stop routes, sequential GEODIST calls over a temporary geo key compute leg-by-leg breakdowns efficiently.
