# How to Monitor RPKI Validation Status for IPv6 Prefixes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RPKI, IPv6, Monitoring, BGP, Routing Security

Description: Learn how to continuously monitor RPKI validation status for your IPv6 prefixes using validators, APIs, and observability tools.

## Why Monitor RPKI?

RPKI validation status can change unexpectedly: ROAs expire, new conflicting ROAs appear, or validator software loses sync with repositories. Continuous monitoring ensures your IPv6 prefixes remain VALID and reachable.

## Monitoring with Routinator HTTP API

Routinator exposes a built-in HTTP API for querying validation status:

```bash
# Start Routinator with HTTP API enabled
routinator server --http [::]:8323 --rtr [::]:3323

# Query validation status for a specific prefix
curl "http://[::1]:8323/api/v1/validity/AS64496/2001:db8::/32"

# Example response:
# {
#   "validated_route": {
#     "route": {"prefix": "2001:db8::/32", "origin_asn": "AS64496"},
#     "validity": {
#       "state": "valid",
#       "description": "At least one VRP Matches the Route Prefix"
#     }
#   }
# }
```

## Building a Monitoring Script

```python
import requests
import json
import sys
from datetime import datetime

# Configuration
ROUTINATOR_API = "http://localhost:8323"
PREFIXES_TO_MONITOR = [
    {"prefix": "2001:db8::/32", "asn": "AS64496"},
    {"prefix": "2001:db8:1::/48", "asn": "AS64496"},
]

def check_rpki_status(prefix, asn):
    """Check RPKI validity for a prefix/ASN pair."""
    url = f"{ROUTINATOR_API}/api/v1/validity/{asn}/{prefix}"
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        data = response.json()
        state = data["validated_route"]["validity"]["state"]
        return state
    except Exception as e:
        return f"error: {e}"

def main():
    timestamp = datetime.utcnow().isoformat()
    all_valid = True

    for entry in PREFIXES_TO_MONITOR:
        state = check_rpki_status(entry["prefix"], entry["asn"])
        status = "OK" if state == "valid" else "ALERT"
        if state != "valid":
            all_valid = False
        print(f"[{timestamp}] {status} {entry['prefix']} ({entry['asn']}): {state}")

    sys.exit(0 if all_valid else 1)

if __name__ == "__main__":
    main()
```

## Prometheus Metrics Integration

Expose RPKI validation metrics for Prometheus scraping:

```python
from prometheus_client import Gauge, start_http_server
import time, requests

# Define metrics
rpki_validity = Gauge(
    'rpki_prefix_validity',
    'RPKI validation state (1=valid, 0=invalid, -1=not-found)',
    ['prefix', 'asn']
)

STATE_MAP = {"valid": 1, "invalid": 0, "not-found": -1}

def update_metrics():
    prefixes = [
        ("2001:db8::/32", "AS64496"),
        ("2001:db8:1::/48", "AS64496"),
    ]
    for prefix, asn in prefixes:
        url = f"http://localhost:8323/api/v1/validity/{asn}/{prefix}"
        try:
            resp = requests.get(url, timeout=5)
            state = resp.json()["validated_route"]["validity"]["state"]
            value = STATE_MAP.get(state, -2)
        except Exception:
            value = -2
        rpki_validity.labels(prefix=prefix, asn=asn).set(value)

if __name__ == "__main__":
    # Start metrics server on IPv6
    start_http_server(8000, addr="::")
    while True:
        update_metrics()
        time.sleep(60)
```

## Grafana Dashboard Query

```
# PromQL: Alert when any prefix is not valid
rpki_prefix_validity != 1

# Count of valid prefixes
count(rpki_prefix_validity == 1)
```

## Monitoring ROA Expiry

ROAs expire when their certificate expires. Check expiry proactively:

```bash
# Check ROA expiry via RIPE Stats API
curl "https://stat.ripe.net/data/rpki-validation/data.json?resource=AS64496&prefix=2001:db8::/32" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(d['data']['validating_roas'], indent=2))"
```

## Monitoring with OneUptime

Configure [OneUptime](https://oneuptime.com) HTTP monitors to poll your Routinator API endpoint for each critical IPv6 prefix. Set up alerts that fire when the validation state changes from "valid" to any other state.

## Conclusion

RPKI validation monitoring requires querying your validator API, exporting metrics to Prometheus, and alerting on state changes. Pay particular attention to ROA expiry dates and validator cache synchronization to avoid unexpected routing issues.
