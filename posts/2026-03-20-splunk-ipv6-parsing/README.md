# How to Parse IPv6 Addresses in Splunk

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Splunk, IPv6, SIEM, Log Parsing, SPL, Security Analytics

Description: Configure Splunk to correctly parse, normalize, and search IPv6 addresses in logs including field extractions, regex patterns, and lookup tables for IPv6 analysis.

## IPv6 Parsing Challenges in Splunk

IPv6 addresses appear in multiple formats in logs:
- Full: `2001:0db8:0000:0000:0000:0000:0000:0001`
- Compressed: `2001:db8::1`
- Mixed: `::ffff:192.168.1.1` (IPv4-mapped)
- With brackets: `[2001:db8::1]:443`
- With port: `2001:db8::1.443` (Cisco format)

Splunk's automatic field extraction often misidentifies IPv6 as multiple fields or fails entirely.

## Props.conf: IPv6 Field Extraction

```ini
# /opt/splunk/etc/system/local/props.conf
# Extract IPv6 addresses from common log formats

[syslog]
# Extract source IPv6 from firewall logs
EXTRACT-src_ipv6 = (?:SRC=|src=|source=|from\s+)(?P<src_ip>[0-9a-fA-F:]+:[0-9a-fA-F:]+)

# Extract destination IPv6
EXTRACT-dst_ipv6 = (?:DST=|dst=|dest=|destination=)(?P<dst_ip>[0-9a-fA-F:]+:[0-9a-fA-F:]+)

# Extract IPv6 from nginx access logs
[nginx_access]
EXTRACT-client_ipv6 = ^(?P<client_ip>[0-9a-fA-F:]+:[0-9a-fA-F:]+)\s

# Extract from Apache combined log (handles both IPv4 and IPv6)
[access_combined]
EXTRACT-remote_host = ^(?P<remote_host>\[?[0-9a-fA-F:.]+\]?)\s
```

## Transforms.conf: IPv6 Normalization

```ini
# /opt/splunk/etc/system/local/transforms.conf
# Normalize IPv6 addresses to compressed form

[normalize_ipv6]
INGEST_EVAL = src_ip=if(match(src_ip, "^[0-9a-fA-F:]+:[0-9a-fA-F:]+$"), src_ip, src_ip)

# Lookup table for IPv6 prefix classification
[ipv6_prefix_lookup]
filename = ipv6_prefixes.csv
case_sensitive_match = false
```

```csv
# /opt/splunk/etc/apps/network_security/lookups/ipv6_prefixes.csv
prefix,type,description
2001:db8::/32,documentation,RFC 3849 documentation prefix
fc00::/7,ula,Unique Local Address
fe80::/10,link-local,Link-Local
ff00::/8,multicast,Multicast
2001::/32,teredo,Teredo tunnel
2002::/16,6to4,6to4 tunnel
::1/128,loopback,Loopback
```

## SPL: IPv6 Search Patterns

```
| SPL queries for IPv6 analysis

-- Search for all events with IPv6 source addresses
index=firewall
| regex src_ip="^[0-9a-fA-F:]+:[0-9a-fA-F:]+$"
| stats count by src_ip, action
| sort -count

-- Extract IPv6 /64 prefix (first 64 bits = 4 groups)
index=firewall
| rex field=src_ip "^(?P<src_prefix64>(?:[0-9a-fA-F]{0,4}:){4})"
| stats dc(src_ip) as unique_hosts, count as events by src_prefix64
| sort -events

-- Detect IPv6 addresses from suspicious ranges
index=network
| search src_ip="2001:*" OR src_ip="fe80:*"
| eval addr_type=case(
    match(src_ip, "^fe80:"), "link-local",
    match(src_ip, "^fc"), "ula",
    match(src_ip, "^ff"), "multicast",
    match(src_ip, "^2001:db8:"), "documentation",
    true(), "global"
)
| stats count by addr_type, src_ip
```

## SPL: IPv6 Subnet Matching

```
| Splunk doesn't natively support IPv6 CIDR matching
| Use cidrmatch() — it works for IPv6 in Splunk 8.x+

index=firewall
| where cidrmatch("2001:db8::/32", src_ip)
| stats count by src_ip, dst_ip, action

-- Find all traffic from RFC 4193 ULA range
index=network
| where cidrmatch("fc00::/7", src_ip)
| stats count by src_ip, dst_ip
| sort -count

-- Traffic from multiple subnets
index=firewall
| eval in_corp=cidrmatch("2001:db8:corp::/48", src_ip)
| eval in_dmz=cidrmatch("2001:db8:dmz::/48", src_ip)
| search in_corp=1 OR in_dmz=1
| stats count by src_ip, in_corp, in_dmz
```

## IPv6 Address Expansion for Normalization

```
| Splunk eval: expand compressed IPv6 for consistent lookup

index=network
| eval normalized_src=lower(src_ip)
-- Remove brackets
| eval normalized_src=replace(normalized_src, "[\[\]]", "")
-- Extract just the IP if port is appended
| rex field=normalized_src "^(?P<normalized_src>[0-9a-fA-F:]+)"

-- Use Python script to normalize IPv6 (custom command)
index=network src_ip="*:*"
| eval src_ip=src_ip
| script normalize_ipv6.py src_ip
| stats count by src_ip
```

```python
# /opt/splunk/etc/apps/network_security/bin/normalize_ipv6.py
# Custom Splunk command for IPv6 normalization
import sys
import ipaddress
from splunklib.searchcommands import dispatch, StreamingCommand, Configuration, Option

@Configuration()
class NormalizeIPv6Command(StreamingCommand):
    field = Option(require=True)

    def stream(self, records):
        for record in records:
            addr = record.get(self.field, '')
            try:
                normalized = str(ipaddress.ip_address(addr))
                record[self.field] = normalized
            except ValueError:
                pass
            yield record

dispatch(NormalizeIPv6Command, sys.argv, sys.stdin, sys.stdout, __name__)
```

## Dashboard: IPv6 Traffic Overview

```xml
<!-- Splunk dashboard panel for IPv6 traffic -->
<panel>
  <title>IPv6 Traffic by Address Type</title>
  <chart>
    <search>
      <query>
        index=firewall
        | where cidrmatch("::/0", src_ip) AND NOT cidrmatch("::ffff:0:0/96", src_ip)
        | eval addr_type=case(
            match(src_ip, "^fe80:"), "Link-Local",
            match(src_ip, "^fc"), "ULA",
            match(src_ip, "^ff"), "Multicast",
            true(), "Global"
          )
        | timechart span=1h count by addr_type
      </query>
    </search>
    <option name="charting.chart">area</option>
  </chart>
</panel>
```

## Conclusion

Splunk IPv6 parsing requires custom field extractions in `props.conf` because auto-extraction often fails with colons and compressed notation. Use regex patterns matching `[0-9a-fA-F:]+:[0-9a-fA-F:]+` for basic IPv6 extraction. Splunk's `cidrmatch()` function (version 8.x+) supports IPv6 CIDR matching for subnet filtering. For normalization, create a custom streaming command using Python's `ipaddress.ip_address()` to expand compressed addresses to full form. Store prefix classification in a CSV lookup table and join at search time to enrich IPv6 addresses with type information.
