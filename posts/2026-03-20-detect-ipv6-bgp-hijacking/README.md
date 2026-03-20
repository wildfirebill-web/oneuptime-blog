# How to Detect IPv6 BGP Hijacking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, IPv6, Security, Monitoring, Routing

Description: Practical techniques for detecting BGP route hijacking of your IPv6 prefixes using RPKI, route monitoring services, and traffic analysis.

## What is BGP Hijacking?

BGP hijacking occurs when an attacker announces your IPv6 prefixes from an unauthorized AS, diverting your traffic. Detection requires monitoring BGP global routing tables and comparing them against what you authorized.

## Detection Method 1: RPKI Validation

The most reliable automated defense is RPKI - any announcement from an unauthorized AS appears as INVALID:

```bash
# Check if your prefix is currently announced with correct origin

curl "https://stat.ripe.net/data/rpki-validation/data.json?resource=AS64496&prefix=2001:db8::/32"

# Use the RIPE BGPmon API to check for unexpected origins
curl "https://stat.ripe.net/data/routing-status/data.json?resource=2001:db8::/32"
```

## Detection Method 2: RIPE RIS BGP Monitoring

RIPE Routing Information Service (RIS) collects BGP data from route collectors worldwide:

```python
import requests
import json

def check_bgp_origins(prefix):
    """Check all ASes currently announcing a prefix via RIPE RIS."""
    url = "https://stat.ripe.net/data/routing-status/data.json"
    params = {"resource": prefix}

    response = requests.get(url, params=params, timeout=30)
    data = response.json()

    origins = data.get("data", {}).get("origins", [])
    print(f"Current announced origins for {prefix}:")
    for origin in origins:
        asn = origin.get("origin", "Unknown")
        visibility = origin.get("visibility", 0)
        print(f"  AS{asn}: {visibility:.1f}% visibility")

    return origins

# Check your prefix
origins = check_bgp_origins("2001:db8::/32")

# Alert if unexpected ASN is announcing
authorized_asns = {"64496", "64497"}
for origin in origins:
    asn = str(origin.get("origin", ""))
    if asn not in authorized_asns:
        print(f"ALERT: Unauthorized AS{asn} announcing your prefix!")
```

## Detection Method 3: BGPStream

BGPStream is a framework for real-time BGP data processing:

```python
# Install: pip install pybgpstream
from pybgpstream import BGPStream

stream = BGPStream(
    from_time="2026-03-19 00:00:00",
    until_time="2026-03-19 01:00:00",
    collectors=["rrc00", "rrc01", "route-views2"],
    record_type="updates"
)

stream.add_filter("prefix", "2001:db8::/32")

for rec in stream.records():
    for elem in rec:
        if elem.type in ("A", "W"):  # Announcement or Withdrawal
            prefix = elem.fields.get("prefix", "")
            as_path = elem.fields.get("as-path", "")
            peer_asn = elem.peer_asn
            origin_asn = as_path.split()[-1] if as_path else "unknown"
            print(f"{elem.type} | {prefix} | path: {as_path} | origin: {origin_asn}")
```

## Detection Method 4: Traceroute-Based Detection

If you control both endpoints, regular traceroutes reveal unexpected path changes:

```bash
# Traceroute to your own IPv6 prefix from an external vantage point
traceroute6 -n 2001:db8::1

# Or use RIPE Atlas for global vantage points
# Atlas probe measurement can be created via API

# Alert if AS path changes unexpectedly
# Compare hop 1,2,3 ASNs with known expected path
```

## Automated Alert Script

```bash
#!/bin/bash
# bgp-hijack-check.sh - Run every 5 minutes via cron

PREFIX="2001:db8::/32"
AUTHORIZED_ASN="64496"
ALERT_EMAIL="noc@example.com"

# Query RIPE stat for current origin
ORIGINS=$(curl -s "https://stat.ripe.net/data/routing-status/data.json?resource=$PREFIX" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for o in data.get('data', {}).get('origins', []):
    print(o.get('origin', ''))
")

# Check each origin against authorized list
for asn in $ORIGINS; do
  if [ "$asn" != "$AUTHORIZED_ASN" ]; then
    echo "ALERT: BGP Hijack detected! AS$asn announcing $PREFIX" | \
      mail -s "BGP Hijack Alert" $ALERT_EMAIL
  fi
done
```

## Monitoring with OneUptime

Use [OneUptime](https://oneuptime.com) to monitor external reachability of your IPv6 addresses from multiple geographic locations. A sudden change in ping latency or packet loss from specific regions often indicates a BGP hijack in progress.

## Conclusion

IPv6 BGP hijacking detection requires RPKI deployment (most effective), BGP route collector monitoring (RIPE RIS, BGPStream), and regular external reachability testing. Combine all three approaches and automate alerts for rapid incident response.
