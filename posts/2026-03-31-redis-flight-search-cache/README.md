# How to Build a Flight Search Cache with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Travel, Cache

Description: Cache expensive flight search results in Redis to deliver instant responses for repeat queries while keeping pricing fresh with short TTLs and smart invalidation.

---

Flight search is one of the most compute-intensive operations in travel - a single search may query dozens of airlines, run fare combination algorithms, and apply ancillary fee calculations. Caching these results in Redis means repeat searches (same route, same dates) return in milliseconds instead of seconds.

## Caching Strategy

Flight prices change frequently, but not on every second. A TTL of 5-10 minutes is a good balance between freshness and cache efficiency. Cache key design must capture all search parameters that affect results.

## Setup

```python
import redis
import json
import time
import hashlib

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

FLIGHT_PREFIX = "flight:search"
FARE_PREFIX = "flight:fare"
POPULAR_ROUTES_KEY = "flight:popular_routes"
SEARCH_TTL = 300       # 5 minutes
FARE_TTL = 60          # 1 minute for individual fare quotes
```

## Building Cache Keys

```python
def search_cache_key(origin: str, destination: str, depart_date: str,
                     return_date: str = None, passengers: int = 1,
                     cabin: str = "economy") -> str:
    params = f"{origin.upper()}:{destination.upper()}:{depart_date}"
    if return_date:
        params += f":{return_date}"
    params += f":{passengers}:{cabin}"
    return f"{FLIGHT_PREFIX}:{hashlib.md5(params.encode()).hexdigest()}"

def readable_key(origin: str, destination: str, depart_date: str) -> str:
    return f"{FLIGHT_PREFIX}:{origin.upper()}-{destination.upper()}:{depart_date}"
```

## Caching Search Results

```python
def get_flight_results(origin: str, destination: str, depart_date: str,
                        return_date: str = None, passengers: int = 1,
                        cabin: str = "economy") -> dict:
    key = search_cache_key(origin, destination, depart_date, return_date, passengers, cabin)
    cached = r.get(key)

    if cached:
        result = json.loads(cached)
        result["cached"] = True
        result["cache_age_seconds"] = int(time.time()) - result["generated_at"]
        return result

    # Cache miss - call flight search engine
    results = search_flights_from_gds(origin, destination, depart_date, return_date, passengers, cabin)
    results["generated_at"] = int(time.time())
    results["cached"] = False

    r.setex(key, SEARCH_TTL, json.dumps(results))

    # Track popular routes
    route = f"{origin.upper()}-{destination.upper()}"
    r.zincrby(POPULAR_ROUTES_KEY, 1, route)

    return results

def search_flights_from_gds(origin, destination, depart_date, return_date, passengers, cabin) -> dict:
    # Replace with actual GDS/NDC API call (Amadeus, Sabre, Travelport)
    return {
        "origin": origin,
        "destination": destination,
        "depart_date": depart_date,
        "flights": [
            {
                "flight_number": "UA123",
                "depart_time": "08:00",
                "arrive_time": "11:30",
                "duration_minutes": 210,
                "stops": 0,
                "price_usd": 349.00,
                "seats_available": 14,
                "cabin": cabin
            },
            {
                "flight_number": "DL456",
                "depart_time": "10:15",
                "arrive_time": "14:00",
                "duration_minutes": 225,
                "stops": 0,
                "price_usd": 299.00,
                "seats_available": 7,
                "cabin": cabin
            }
        ]
    }
```

## Locking Fare During Checkout

```python
def lock_fare(session_id: str, flight_number: str, depart_date: str,
              fare_usd: float, passenger_count: int) -> bool:
    lock_key = f"{FARE_PREFIX}:lock:{flight_number}:{depart_date}:{session_id}"
    fare_data = json.dumps({
        "flight_number": flight_number,
        "depart_date": depart_date,
        "fare_usd": fare_usd,
        "passenger_count": passenger_count,
        "locked_at": int(time.time())
    })
    # Hold fare for 15 minutes during checkout
    return bool(r.set(lock_key, fare_data, ex=900, nx=True))

def verify_fare_lock(session_id: str, flight_number: str, depart_date: str) -> dict | None:
    lock_key = f"{FARE_PREFIX}:lock:{flight_number}:{depart_date}:{session_id}"
    raw = r.get(lock_key)
    return json.loads(raw) if raw else None
```

## Popular Routes

```python
def get_popular_routes(top_n: int = 20) -> list:
    return [
        {"route": route, "searches": int(count)}
        for route, count in r.zrange(POPULAR_ROUTES_KEY, 0, top_n - 1, desc=True, withscores=True)
    ]
```

## Price Alert Cache

```python
def set_price_alert(user_id: str, origin: str, destination: str, target_price: float):
    alert_key = f"flight:alert:{user_id}:{origin}-{destination}"
    r.hset(alert_key, mapping={
        "target_price": target_price,
        "origin": origin,
        "destination": destination,
        "created_at": int(time.time())
    })
    # Add to monitored routes set
    r.sadd(f"flight:monitored_routes:{origin}-{destination}", user_id)

def check_price_alerts(origin: str, destination: str, current_price: float, depart_date: str):
    watchers = r.smembers(f"flight:monitored_routes:{origin}-{destination}")
    for user_id in watchers:
        alert = r.hgetall(f"flight:alert:{user_id}:{origin}-{destination}")
        if alert and current_price <= float(alert["target_price"]):
            r.publish("flight:price_alerts", json.dumps({
                "user_id": user_id, "origin": origin,
                "destination": destination, "current_price": current_price,
                "depart_date": depart_date
            }))
```

## Summary

A Redis flight search cache uses MD5-hashed parameter-based keys for consistent cache hits across equivalent searches, short 5-minute TTLs to balance price freshness with cache efficiency, and SET NX fare locks during checkout to protect quoted prices. Popular routes and price alerts are tracked with sorted sets and sets for secondary use cases.
