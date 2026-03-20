# How to Implement IP-Based Geolocation in REST APIs Using IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: REST API, IPv4, Geolocation, Python, Node.js, Networking

Description: Learn how to implement IPv4 address-based geolocation in REST APIs using external APIs and local databases like MaxMind GeoLite2, with caching and fallback strategies.

## Using ipapi.co (Free, No Install)

```python
import httpx
from flask import Flask, request, jsonify

app = Flask(__name__)

async def geolocate(ip: str) -> dict:
    async with httpx.AsyncClient(timeout=3.0) as client:
        resp = await client.get(f"https://ipapi.co/{ip}/json/")
        if resp.status_code == 200:
            data = resp.json()
            return {
                "ip":      data.get("ip"),
                "country": data.get("country_name"),
                "city":    data.get("city"),
                "region":  data.get("region"),
                "lat":     data.get("latitude"),
                "lon":     data.get("longitude"),
                "org":     data.get("org"),
            }
    return {"error": "lookup failed"}
```

## MaxMind GeoLite2 (Local Database — No API Calls)

```bash
# Install geoip2 library
pip install geoip2

# Download GeoLite2-City.mmdb from https://dev.maxmind.com/geoip/geolite2-free-geolocation-data
```

```python
import geoip2.database
from flask import Flask, request, jsonify

app = Flask(__name__)

# Load once at startup
_reader = geoip2.database.Reader("/opt/GeoLite2-City.mmdb")

def geolocate_local(ip: str) -> dict:
    try:
        response = _reader.city(ip)
        return {
            "ip":         ip,
            "country":    response.country.name,
            "country_iso":response.country.iso_code,
            "city":       response.city.name,
            "region":     response.subdivisions.most_specific.name,
            "lat":        response.location.latitude,
            "lon":        response.location.longitude,
            "timezone":   response.location.time_zone,
        }
    except geoip2.errors.AddressNotFoundError:
        return {"ip": ip, "error": "not found"}
    except Exception as e:
        return {"ip": ip, "error": str(e)}

@app.get("/api/geo")
def geo_endpoint():
    xff = request.headers.get("X-Forwarded-For")
    ip  = xff.split(",")[0].strip() if xff else request.remote_addr
    return jsonify(geolocate_local(ip))
```

## Node.js with @maxmind/geoip2-node

```javascript
const { WebServiceClient } = require("@maxmind/geoip2-node");
const Reader = require("@maxmind/geoip2-node").Reader;
const express = require("express");

const app = express();
app.set("trust proxy", 1);

let dbReader;

(async () => {
    // Open the local MMDB file
    dbReader = await Reader.open("/opt/GeoLite2-City.mmdb");
})();

app.get("/api/geo", async (req, res) => {
    const ip = req.ip;
    try {
        const result = await dbReader.city(ip);
        res.json({
            ip,
            country:  result.country?.names?.en,
            city:     result.city?.names?.en,
            lat:      result.location?.latitude,
            lon:      result.location?.longitude,
            timezone: result.location?.timeZone,
        });
    } catch (err) {
        res.status(404).json({ ip, error: "not found" });
    }
});

app.listen(3000);
```

## Caching Geolocation Results

```python
from functools import lru_cache
import geoip2.database

_reader = geoip2.database.Reader("/opt/GeoLite2-City.mmdb")

@lru_cache(maxsize=10_000)
def cached_geolocate(ip: str) -> tuple:
    """Cache up to 10 000 IP lookups in memory."""
    try:
        r = _reader.city(ip)
        return (r.country.iso_code, r.country.name, r.city.name)
    except Exception:
        return (None, None, None)
```

## Conclusion

For high-volume APIs, use a local MaxMind GeoLite2 MMDB database — lookups take microseconds and require no network call. External services (ipapi.co, ip-api.com) are convenient for low traffic but add latency and rate limits. Cache results aggressively with `lru_cache` or Redis since IP-to-geo mappings change rarely. Always extract the real client IP using proxy-aware logic before calling the geolocation function, and skip private/loopback addresses which won't have geolocation data.
