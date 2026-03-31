# How to Implement Geo-Based Content Delivery with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Geospatial, Content Delivery, CDN, Edge

Description: Use Redis Geospatial to route users to the nearest content server, edge node, or CDN PoP for minimal latency content delivery.

---

Routing users to the nearest content delivery node reduces latency and improves performance. Redis Geo stores server coordinates and finds the closest node to any user in one command, making it a fast routing layer for edge content delivery.

## Registering Edge Nodes

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def register_edge_node(node_id: str, lat: float, lon: float,
                       region: str, capacity: int):
    pipe = r.pipeline()
    pipe.geoadd("cdn:nodes:geo", [lon, lat, node_id])
    pipe.hset(f"cdn:node:{node_id}", mapping={
        "node_id": node_id,
        "region": region,
        "lat": lat, "lon": lon,
        "capacity": capacity,
        "current_load": 0,
        "status": "active",
    })
    pipe.execute()

def update_node_load(node_id: str, load_pct: float):
    r.hset(f"cdn:node:{node_id}", "current_load", load_pct)
    # Remove overloaded nodes from routing
    if load_pct > 90:
        r.zrem("cdn:nodes:geo", node_id)
    elif load_pct <= 90:
        # Re-add if recovered
        meta = r.hgetall(f"cdn:node:{node_id}")
        if meta:
            r.geoadd("cdn:nodes:geo", [
                float(meta["lon"]), float(meta["lat"]), node_id
            ])
```

## Routing a User to the Nearest Node

```python
def get_nearest_node(user_lat: float, user_lon: float,
                     max_nodes: int = 3) -> list:
    results = r.geosearch(
        "cdn:nodes:geo",
        longitude=user_lon, latitude=user_lat,
        radius=20000, unit="km",  # Global search
        sort="ASC", count=max_nodes,
        withdist=True,
    )

    nodes = []
    pipe = r.pipeline()
    for node_id, _ in results:
        pipe.hgetall(f"cdn:node:{node_id}")
    node_metas = pipe.execute()

    for (node_id, dist), meta in zip(results, node_metas):
        if meta.get("status") == "active":
            nodes.append({
                "node_id": node_id,
                "distance_km": round(float(dist), 2),
                "region": meta.get("region"),
                "load_pct": float(meta.get("current_load", 0)),
            })
    return nodes

def route_request(user_lat: float, user_lon: float) -> str:
    candidates = get_nearest_node(user_lat, user_lon, max_nodes=3)
    if not candidates:
        return "origin"

    # Pick lowest-loaded from top 3 nearest
    best = min(candidates, key=lambda x: x["load_pct"])
    return best["node_id"]
```

## Content Geo-Restrictions

```python
def is_content_allowed(content_id: str, user_lat: float,
                        user_lon: float) -> bool:
    allowed_regions_key = f"content:{content_id}:allowed_regions"
    if not r.exists(allowed_regions_key):
        return True  # No restrictions

    # Find which region the user is in
    nearest = r.geosearch(
        "regions:geo",
        longitude=user_lon, latitude=user_lat,
        radius=500, unit="km",
        sort="ASC", count=1,
    )
    if not nearest:
        return False

    region_id = nearest[0]
    return r.sismember(allowed_regions_key, region_id)
```

## Node Health Status

```python
def get_all_nodes_status() -> list:
    node_ids = r.zrange("cdn:nodes:geo", 0, -1)
    pipe = r.pipeline()
    for nid in node_ids:
        pipe.hgetall(f"cdn:node:{nid}")
    return pipe.execute()
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to monitor each edge node's health endpoint and auto-remove unresponsive nodes from the Redis geo routing index.

```bash
redis-cli ZCARD cdn:nodes:geo
redis-cli HGETALL cdn:node:us-east-1
```

## Summary

Redis Geo turns edge node routing into a two-step operation: GEOSEARCH finds the nearest nodes, and a load check selects the best candidate. Overloaded nodes are removed from the geo index automatically, ensuring requests are never routed to struggling nodes. This pattern works equally well for CDN PoP selection, microservice instance routing, and regional database replica selection.
