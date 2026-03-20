# How to Normalize IPv6 Addresses in Log Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Log Normalization, SIEM, ECS, CEF, LEEF, Data Pipelines

Description: Normalize IPv6 addresses across different log formats and representations to enable consistent SIEM analysis, correlation, and threat detection.

## Why IPv6 Normalization Matters

The same IPv6 address can appear in dozens of different formats across log sources:
- `2001:0db8:0000:0000:0000:0000:0000:0001` (full)
- `2001:db8::1` (compressed)
- `::ffff:192.168.1.1` (IPv4-mapped)
- `[2001:db8::1]` (bracketed, URL syntax)
- `[2001:db8::1]:443` (with port)
- `2001:db8::1:443` (Cisco/some syslog)
- `2001:db8::1.443` (some syslog daemons)

Without normalization, correlation queries miss matches and dashboards undercount events.

## Canonical IPv6 Form

The canonical (normalized) form uses RFC 5952 compressed notation:
- Lowercase hexadecimal
- Longest run of zeros replaced with `::`
- Leading zeros in groups removed
- Example: `2001:db8::1` (not `2001:0DB8:0000::0001`)

## Python: IPv6 Normalization Library

```python
import ipaddress
import re
from typing import Optional

def normalize_ipv6(addr: str) -> Optional[str]:
    """
    Normalize an IPv6 address to RFC 5952 canonical form.
    Returns None if input is not a valid IPv6 address.
    """
    # Strip brackets and port
    addr = addr.strip()

    # Handle [2001:db8::1]:port format
    bracket_match = re.match(r'^\[([^\]]+)\](?::\d+)?$', addr)
    if bracket_match:
        addr = bracket_match.group(1)

    # Handle 2001:db8::1.port format (dot-separated port)
    dot_port_match = re.match(r'^([0-9a-fA-F:]+:[0-9a-fA-F:]+)\.(\d+)$', addr)
    if dot_port_match:
        addr = dot_port_match.group(1)

    try:
        return str(ipaddress.ip_address(addr))
    except ValueError:
        return None

def extract_prefix(addr: str, prefix_len: int = 64) -> Optional[str]:
    """Extract the /N prefix from an IPv6 address."""
    normalized = normalize_ipv6(addr)
    if not normalized:
        return None
    try:
        net = ipaddress.ip_network(f"{normalized}/{prefix_len}", strict=False)
        return str(net.network_address)
    except ValueError:
        return None

def classify_ipv6(addr: str) -> str:
    """Classify an IPv6 address type."""
    normalized = normalize_ipv6(addr)
    if not normalized:
        return "invalid"
    ip = ipaddress.ip_address(normalized)
    if ip.is_loopback:           return "loopback"
    if ip.is_link_local:         return "link-local"
    if ip.is_private:            return "ula"
    if ip.is_multicast:          return "multicast"
    if ip.ipv4_mapped:           return "ipv4-mapped"
    if str(ip).startswith("2002"): return "6to4"
    if str(ip).startswith("2001:0:"): return "teredo"
    return "global"

# Test

tests = [
    "2001:0db8:0000:0000:0000:0000:0000:0001",
    "[2001:db8::1]:443",
    "::ffff:192.168.1.1",
    "FE80::1",
]
for t in tests:
    print(f"{t!r:50} → {normalize_ipv6(t)!r} ({classify_ipv6(t)})")
```

## Logstash: IPv6 Normalization Filter

```ruby
# /etc/logstash/conf.d/normalize-ipv6.conf

filter {
  # Normalize source IP
  if [source][ip] {
    ruby {
      code => '
        require "ipaddr"
        begin
          ip = event.get("[source][ip]")
          # Strip brackets and port
          ip = ip.gsub(/^\[/, "").gsub(/\].*$/, "")
          normalized = IPAddr.new(ip).to_s
          event.set("[source][ip]", normalized)
          # Set address type
          addr = IPAddr.new(normalized)
          type = if addr.loopback? then "loopback"
                 elsif addr.link_local? then "link-local"
                 elsif normalized.start_with?("fc", "fd") then "ula"
                 elsif addr.multicast? then "multicast"
                 else "global" end
          event.set("[source][ip_type]", type)
          # Extract /64 prefix
          prefix64 = IPAddr.new(normalized).mask(64).to_s
          event.set("[source][prefix64]", prefix64)
        rescue => e
          # Not an IPv6 address - leave as-is
        end
      '
    }
  }
}
```

## CEF/LEEF Log Normalization

```python
import ipaddress
import re

def parse_cef_ipv6(cef_event: str) -> dict:
    """
    Parse CEF (Common Event Format) log and normalize IPv6 fields.
    """
    fields = {}

    # Extract src/dst from CEF extension
    # CEF uses 'src' for source, 'dst' for destination
    for key, pattern in [
        ('src', r'src=([^\s|]+)'),
        ('dst', r'dst=([^\s|]+)'),
        ('sourceAddress', r'sourceAddress=([^\s|]+)'),
        ('destinationAddress', r'destinationAddress=([^\s|]+)'),
    ]:
        match = re.search(pattern, cef_event)
        if match:
            raw = match.group(1)
            # Normalize IPv6 addresses
            clean = re.sub(r'^\[|\].*$', '', raw)
            try:
                fields[key] = str(ipaddress.ip_address(clean))
                fields[f"{key}_type"] = "ipv6" if ":" in fields[key] else "ipv4"
            except ValueError:
                fields[key] = raw

    return fields

# Test CEF parsing
cef = "CEF:0|Firewall|FW|1.0|100|Deny|5|src=2001:0DB8::0001 dst=[2001:db8:1::10]:443"
result = parse_cef_ipv6(cef)
print(result)
# {'src': '2001:db8::1', 'src_type': 'ipv6', 'dst': '2001:db8:1::10', 'dst_type': 'ipv6'}
```

## Normalization Pipeline Validation

```bash
#!/bin/bash
# test-normalization.sh - Validate IPv6 normalization output

python3 << 'EOF'
import ipaddress

test_cases = [
    # input                           expected output
    ("2001:0db8:0000:0000:0000:0000:0000:0001", "2001:db8::1"),
    ("2001:DB8::1",                             "2001:db8::1"),
    ("::ffff:192.168.1.1",                      "::ffff:192.168.1.1"),
    ("[2001:db8::1]:443",                        "2001:db8::1"),
    ("FE80::1",                                  "fe80::1"),
    ("::1",                                      "::1"),
]

import re

def normalize(addr):
    addr = addr.strip()
    bracket_match = re.match(r'^\[([^\]]+)\]', addr)
    if bracket_match:
        addr = bracket_match.group(1)
    try:
        return str(ipaddress.ip_address(addr))
    except:
        return None

passed = 0
for raw, expected in test_cases:
    result = normalize(raw)
    status = "PASS" if result == expected else "FAIL"
    if status == "PASS":
        passed += 1
    print(f"[{status}] {raw!r:50} → {result!r} (expected {expected!r})")

print(f"\n{passed}/{len(test_cases)} tests passed")
EOF
```

## SIEM Platform: Normalized Field Mapping

```text
# Mapping table: raw field → normalized ECS field

Platform    Raw Field           ECS Field           Normalization
Splunk      src                 source.ip           IPv6 compression
Cisco ASA   %SRC                source.ip           Strip port
iptables    SRC=                source.ip           Strip "SRC="
nginx       $remote_addr        source.ip           Strip brackets
Apache      %h                  source.ip           Strip brackets
CEF         src                 source.ip           ipaddress.ip_address()
LEEF        src                 source.ip           ipaddress.ip_address()
```

## Conclusion

IPv6 log normalization is a prerequisite for effective SIEM correlation. Without it, the same host appears under a dozen different string representations, breaking deduplication, rate counting, and lookup enrichment. The canonical form (RFC 5952 lowercase compressed) is produced by Python's `ipaddress.ip_address()`, Ruby's `IPAddr`, or JavaScript's `new URL()` with IPv6 address parsing. Build normalization into the ingestion pipeline (Logstash filter, Splunk `eval`, or Elastic ingest pipeline) so all downstream analysis works on consistent data. Always extract and store the /64 prefix alongside the full address - it enables meaningful aggregation in large IPv6 deployments.
