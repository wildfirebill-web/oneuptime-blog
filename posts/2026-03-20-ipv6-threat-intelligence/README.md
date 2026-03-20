# How to Use IPv6 Threat Intelligence

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Threat Intelligence, MISP, STIX, IOC, Security Analytics, SIEM

Description: Integrate IPv6 addresses and prefixes into threat intelligence workflows using MISP, STIX 2.1, and SIEM lookups for blocking, detection, and enrichment.

## IPv6 in Threat Intelligence

Threat intelligence for IPv6 includes:
- **IPv6 IOCs**: known malicious /128 addresses
- **Prefix blocklists**: malicious /48 or /32 ranges
- **Reputation scores**: per-prefix risk rating
- **Attack infrastructure**: C2 servers, scanners, proxies
- **Tunnel abuse**: 6to4/Teredo prefixes used for evasion

IPv6 IOC management faces unique challenges: privacy extensions mean individual addresses rotate, so /64 prefix-level blocking is often more effective than /128.

## STIX 2.1: IPv6 Address Indicators

```json
{
    "type": "bundle",
    "id": "bundle--a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "objects": [
        {
            "type": "indicator",
            "spec_version": "2.1",
            "id": "indicator--b2c3d4e5-f6a7-8901-bcde-f12345678901",
            "created": "2026-03-20T00:00:00Z",
            "modified": "2026-03-20T00:00:00Z",
            "name": "Malicious IPv6 Scanner",
            "description": "IPv6 address observed conducting network scanning",
            "pattern": "[ipv6-addr:value = '2001:db8:attacker::1']",
            "pattern_type": "stix",
            "valid_from": "2026-03-20T00:00:00Z",
            "indicator_types": ["malicious-activity"],
            "labels": ["scanning", "reconnaissance"]
        },
        {
            "type": "indicator",
            "spec_version": "2.1",
            "id": "indicator--c3d4e5f6-a7b8-9012-cdef-123456789012",
            "name": "Malicious IPv6 /48 Prefix",
            "description": "IPv6 /48 prefix hosting attack infrastructure",
            "pattern": "[network-traffic:dst_ref.type = 'ipv6-addr' AND network-traffic:dst_ref.value LIKE '2001:db8:malicious::/48']",
            "pattern_type": "stix",
            "valid_from": "2026-03-20T00:00:00Z",
            "indicator_types": ["malicious-activity"]
        }
    ]
}
```

## MISP: Managing IPv6 IOCs

```python
import pymisp
from pymisp import PyMISP, MISPEvent, MISPAttribute

# Connect to MISP instance (via IPv6)
misp = PyMISP(
    url="https://[2001:db8::misp]/",
    key="your-api-key",
    ssl=True
)

# Create event for IPv6 threat
event = MISPEvent()
event.info = "IPv6 Scanning Campaign"
event.threat_level_id = 2  # Medium
event.analysis = 1          # Ongoing
event.distribution = 1      # This community only

# Add IPv6 IOC attributes
event.add_attribute("ip-src", "2001:db8:attacker::1")
event.add_attribute("ip-src", "2001:db8:attacker::2")

# Add prefix as CIDR (network attribute)
event.add_attribute("ip-src", "2001:db8:attacker::/48")

# Add tag
event.add_tag("tlp:amber")
event.add_tag("attack:scanning")

# Publish
misp.add_event(event)

# Search for IPv6 IOCs
results = misp.search(
    type_attribute="ip-src",
    value="2001:db8:attacker::1",
    pythonify=True
)
for ioc in results:
    print(f"IOC: {ioc['value']} - Event: {ioc['event_id']}")
```

## SIEM Lookup: IPv6 Threat List

```bash
# Generate SIEM lookup file from MISP
#!/usr/bin/env python3
# misp-to-siem-lookup.py

import pymisp
import csv
import ipaddress

misp = pymisp.PyMISP(
    url="https://[2001:db8::misp]/",
    key="YOUR_API_KEY"
)

# Fetch all IPv6 IOCs
iocs = misp.search(type_attribute="ip-src", to_ids=True, pythonify=True)

output_file = "/opt/splunk/etc/apps/threat_intel/lookups/ipv6_iocs.csv"

with open(output_file, 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(["ip", "prefix64", "threat_type", "confidence", "source", "event_id"])

    for ioc in iocs:
        ip = ioc.get('value', '')
        try:
            addr = ipaddress.ip_address(ip)
            prefix64 = str(ipaddress.ip_network(f"{ip}/64", strict=False).network_address)
        except ValueError:
            # Try as network
            try:
                net = ipaddress.ip_network(ip, strict=False)
                ip = str(net)
                prefix64 = str(net.network_address)
            except ValueError:
                continue

        writer.writerow([
            ip,
            prefix64,
            ioc.get('category', 'unknown'),
            ioc.get('confidence', 50),
            "MISP",
            ioc.get('event_id', '')
        ])

print(f"Exported {len(iocs)} IPv6 IOCs to {output_file}")
```

## Splunk: IPv6 Threat Intel Enrichment

```
# Enrich firewall events with threat intelligence
index=firewall src_ip="*:*"

| eval prefix64=replace(src_ip,
    "^([0-9a-fA-F:]{0,4}:[0-9a-fA-F:]{0,4}:[0-9a-fA-F:]{0,4}:[0-9a-fA-F:]{0,4}).*",
    "\1::")

| lookup ipv6_iocs.csv ip AS src_ip OUTPUT threat_type, confidence, source
| lookup ipv6_iocs.csv prefix64 AS prefix64
    OUTPUT threat_type AS prefix_threat, confidence AS prefix_confidence

| eval intel_match=if(isnotnull(threat_type), "exact",
    if(isnotnull(prefix_threat), "prefix", "none"))
| where intel_match != "none"
| stats count by src_ip, prefix64, intel_match, threat_type, confidence
| sort -confidence
```

## Automated Blocklist from Threat Intel

```bash
#!/bin/bash
# update-ipv6-blocklist.sh — Apply threat intel as nftables rules

MISP_URL="https://[2001:db8::misp]"
MISP_KEY="YOUR_API_KEY"

# Fetch malicious IPv6 addresses from MISP
MALICIOUS_IPS=$(curl -s -H "Authorization: ${MISP_KEY}" \
    -H "Accept: application/json" \
    "${MISP_URL}/attributes/restSearch" \
    -d '{"type":"ip-src","to_ids":1,"returnFormat":"text"}' | \
    grep -E "^[0-9a-fA-F:]+:[0-9a-fA-F:]+")

# Create nftables set
nft flush set ip6 filter threat_intel_ipv6 2>/dev/null
nft add set ip6 filter threat_intel_ipv6 '{ type ipv6_addr; flags interval; }'

# Add IPs to set
for IP in ${MALICIOUS_IPS}; do
    nft add element ip6 filter threat_intel_ipv6 "{ ${IP} }"
done

# Add blocking rule (if not exists)
nft add rule ip6 filter input ip6 saddr @threat_intel_ipv6 \
    counter log prefix "THREAT_INTEL_BLOCK: " drop

echo "Blocklist updated: $(echo "${MALICIOUS_IPS}" | wc -l) IPv6 addresses"
```

## Conclusion

IPv6 threat intelligence workflows mirror IPv4 but require /64-prefix awareness — individual /128 addresses rotate due to privacy extensions, making prefix-level blocking more durable. Store IPv6 IOCs in STIX 2.1 format using the `ipv6-addr` object type. MISP handles IPv6 attributes natively — use `ip-src` and `ip-dst` attribute types with IPv6 values. Export IOCs to SIEM lookup tables with both the full address and the /64 prefix for enrichment at both levels. Automate blocklist updates from MISP to nftables sets for operational blocking, and run enrichment queries in Splunk or Elastic to identify IOC matches in historical logs.
