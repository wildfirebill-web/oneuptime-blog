# How to Create IPv6 GeoIP Enrichment in Log Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, GeoIP, Log Enrichment, MaxMind, Elasticsearch

Description: Enrich log records with geographic information for IPv6 addresses using MaxMind GeoLite2 databases, covering Elasticsearch ingest pipelines, Logstash, Fluent Bit, and Python implementations.

## Introduction

GeoIP enrichment adds geographic context (country, city, coordinates) to IP addresses in log data. MaxMind's GeoLite2 databases support IPv6 since version 2. This guide covers enrichment at different points in the log pipeline using common tools.

## Step 1: Download MaxMind GeoLite2 Database

```bash
# Register for free at https://www.maxmind.com then:

# Download GeoLite2-City database (supports IPv6)
wget -O /tmp/GeoLite2-City.tar.gz \
  "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-City&license_key=YOUR_KEY&suffix=tar.gz"

mkdir -p /etc/geoip
tar -xzf /tmp/GeoLite2-City.tar.gz -C /tmp/
mv /tmp/GeoLite2-City_*/GeoLite2-City.mmdb /etc/geoip/

# Verify the database handles IPv6
python3 -c "
import geoip2.database
with geoip2.database.Reader('/etc/geoip/GeoLite2-City.mmdb') as r:
    resp = r.city('2001:4860:4860::8888')  # Google Public DNS IPv6
    print(resp.country.name, resp.city.name)
"
```

## Step 2: Elasticsearch Ingest Pipeline for IPv6 GeoIP

```json
PUT _ingest/pipeline/geoip-ipv6-pipeline
{
  "description": "Enrich logs with GeoIP data for IPv6 addresses",
  "processors": [
    {
      "geoip": {
        "field": "client_ip",
        "target_field": "geoip",
        "database_file": "GeoLite2-City.mmdb",
        "ignore_missing": true,
        "ignore_failure": true,
        "properties": ["continent_name", "country_iso_code", "country_name",
                       "city_name", "region_iso_code", "location"]
      }
    },
    {
      "set": {
        "if": "ctx.geoip?.location != null",
        "field": "geoip.location.type",
        "value": "Point"
      }
    }
  ]
}
```

```bash
# Test the pipeline with an IPv6 address
POST _ingest/pipeline/geoip-ipv6-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "client_ip": "2001:4860:4860::8888",
        "message": "Test request"
      }
    }
  ]
}
```

## Step 3: Python GeoIP Enrichment

```python
#!/usr/bin/env python3
# geoip_enrich.py

import geoip2.database
import geoip2.errors
import ipaddress
from functools import lru_cache

class IPv6GeoIPEnricher:
    def __init__(self, db_path: str):
        self.reader = geoip2.database.Reader(db_path)

    @lru_cache(maxsize=4096)
    def lookup(self, ip: str) -> dict:
        """Look up GeoIP data for an IPv4 or IPv6 address."""
        try:
            response = self.reader.city(ip)
            return {
                "country_code": response.country.iso_code,
                "country_name": response.country.name,
                "city": response.city.name,
                "latitude": response.location.latitude,
                "longitude": response.location.longitude,
                "is_anonymous_proxy": response.traits.is_anonymous_proxy,
            }
        except geoip2.errors.AddressNotFoundError:
            return {}
        except Exception as e:
            return {"geoip_error": str(e)}

    def enrich_record(self, record: dict) -> dict:
        """Add GeoIP data to a log record."""
        ip = record.get("client_ip") or record.get("src_ip")
        if not ip:
            return record

        # Normalize
        ip = ip.split('%')[0].strip('[]')

        # Skip private/reserved addresses
        try:
            addr = ipaddress.ip_address(ip)
            if addr.is_private or addr.is_loopback or addr.is_link_local:
                record["geoip"] = {"type": "private"}
                return record
        except ValueError:
            return record

        geo = self.lookup(ip)
        if geo:
            record["geoip"] = geo
        return record

    def close(self):
        self.reader.close()

# Usage
enricher = IPv6GeoIPEnricher("/etc/geoip/GeoLite2-City.mmdb")

# Test with IPv6 addresses
records = [
    {"client_ip": "2001:4860:4860::8888", "path": "/api/test"},  # Google
    {"client_ip": "2400:cb00:2048::1",    "path": "/"},           # Cloudflare
    {"client_ip": "::1",                  "path": "/health"},     # Loopback
]

for record in records:
    enriched = enricher.enrich_record(record)
    print(enriched)

enricher.close()
```

## Step 4: Logstash GeoIP Filter for IPv6

```ruby
# logstash.conf
filter {
  # First normalize the IP address
  mutate {
    gsub => ["client_ip", "\\[", ""]
    gsub => ["client_ip", "\\]", ""]
    gsub => ["client_ip", "%.*$", ""]  # Strip zone ID
  }

  geoip {
    source => "client_ip"
    target => "geoip"
    database => "/etc/geoip/GeoLite2-City.mmdb"
    # GeoIP2 plugin handles IPv6 automatically
  }
}
```

## Step 5: Fluent Bit GeoIP Enrichment

```ini
# Requires fluent-bit built with geoip2 support
[FILTER]
    Name          geoip2
    Match         nginx.*
    Database      /etc/geoip/GeoLite2-City.mmdb
    Lookup_key    client_ip
    Record        country_code  ${country.iso_code}
    Record        city          ${city.names.en}
    Record        latitude      ${location.latitude}
    Record        longitude     ${location.longitude}
```

## Conclusion

MaxMind GeoLite2 databases support IPv6 addresses in the same lookup calls as IPv4, making IPv6 GeoIP enrichment straightforward. Always normalize IPv6 addresses (strip zone IDs, brackets) before lookup, and skip private/link-local/loopback addresses that have no geographic meaning. Cache lookup results with `@lru_cache` or equivalent since many log entries will share the same source IPv6 address. Enrich at collection time (Logstash/Fluent Bit) rather than query time for better performance at scale.
