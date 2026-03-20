# How to Write IPv6 SIEM Correlation Rules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, SIEM, Correlation Rules, Security Analytics, Threat Detection, Splunk, Elastic

Description: Build multi-event IPv6 correlation rules in SIEM platforms to detect complex attacks including lateral movement, exfiltration, and IPv6-specific protocol abuse.

## Correlation vs Detection Rules

| Type | Scope | Example |
|---|---|---|
| Single-event detection | One log entry | RA from unauthorized source |
| Correlation rule | Multiple events over time | Scan → auth attempt → successful login |
| Behavioral baseline | Deviation from normal | Sudden spike in IPv6 prefix count |

Correlation is most valuable for detecting multi-stage attacks that evade single-event rules.

## Correlation Scenario 1: IPv6 Reconnaissance → Exploitation

```
Attack chain:
1. Attacker scans IPv6 /64 prefix (many ICMP probe failures)
2. Finds active host (ICMP reply)
3. Attempts SSH login (multiple failures)
4. Successful login

Correlation: Events 1-4 from same /64 source prefix within 30 minutes
```

```
# Splunk correlation: detect scan-then-exploit from same /64
| tstats count as events
    where index=firewall OR index=auth
    by _time span=30m, source_prefix64, event_type
| eval src_prefix64=replace(src_ip, ":[0-9a-f:]+$", "::")
| stats
    values(event_type) as event_types,
    count(eval(event_type="icmp_drop")) as scan_events,
    count(eval(event_type="ssh_fail")) as ssh_fails,
    count(eval(event_type="ssh_success")) as ssh_success
    by src_prefix64, _time
| where scan_events > 20 AND ssh_fails > 3 AND ssh_success > 0
| eval threat="IPv6_Recon_To_Exploit"
```

## Correlation Scenario 2: IPv6 Lateral Movement

```
# Elastic EQL: detect lateral movement pattern
# Source /64 accesses multiple internal /64 subnets in sequence

sequence by source.prefix64 with maxspan=10m
  [network where network.type == "ipv6" and event.action == "allowed"
   and destination.ip : "2001:db8:host1::/64"
   and source.ip : "2001:db8:external::/48"]
  [network where network.type == "ipv6" and event.action == "allowed"
   and destination.ip : "2001:db8:host2::/64"
   and source.ip : "2001:db8:host1::/64"]
  [network where network.type == "ipv6" and event.action == "allowed"
   and destination.ip : "2001:db8:host3::/64"
   and source.ip : "2001:db8:host2::/64"]
```

## Correlation Scenario 3: IPv6 Data Exfiltration

```
# Splunk: detect unusually large outbound IPv6 transfers

index=netflow network_type=ipv6
| stats
    sum(bytes_out) as total_bytes_out,
    dc(dst_ip) as unique_destinations,
    earliest(_time) as first_seen,
    latest(_time) as last_seen
    by src_ip
| where total_bytes_out > 1073741824  # 1GB
| eval gb_out = round(total_bytes_out / 1073741824, 2)
| eval duration_min = round((last_seen - first_seen) / 60, 1)
| eval mbps = round(total_bytes_out * 8 / (last_seen - first_seen + 1) / 1048576, 2)
| where NOT cidrmatch("2001:db8:backup::/48", src_ip)
| eval threat="IPv6_Data_Exfiltration_Suspect"
| table src_ip, gb_out, unique_destinations, duration_min, mbps
| sort -gb_out
```

## IPv6 Prefix-Based Correlation

```python
#!/usr/bin/env python3
# extract-prefix64.py — Helper for SIEM correlation
# Extracts /64 prefix for grouping related IPv6 addresses

import ipaddress

def get_prefix64(ip_str: str) -> str:
    """Get the /64 prefix of an IPv6 address."""
    try:
        ip = ipaddress.ip_address(ip_str)
        if isinstance(ip, ipaddress.IPv6Address):
            net = ipaddress.ip_network(f"{ip}/64", strict=False)
            return str(net.network_address)
    except ValueError:
        pass
    return ip_str

# In Logstash/Elasticsearch ingest pipeline:
# Add prefix64 as a derived field at ingestion time
# Reduces correlation complexity — group by prefix64 instead of full /128

# Example: compute_prefix64 ingest processor
ingest_pipeline = {
    "processors": [
        {
            "script": {
                "lang": "painless",
                "source": """
                    if (ctx?.source?.ip != null && ctx.source.ip.contains(":")) {
                        // IPv6: extract first 64 bits
                        String[] parts = ctx.source.ip.split(":");
                        if (parts.length >= 4) {
                            ctx.source.prefix64 = parts[0] + ":" + parts[1] + ":" + parts[2] + ":" + parts[3] + "::";
                        }
                    }
                """
            }
        }
    ]
}
```

## Baseline Deviation Correlation

```
# Splunk: detect unusual new IPv6 /64 prefixes

| tstats count as current_count
    where index=firewall earliest=-1h latest=now
    by src_prefix64

| join type=left src_prefix64 [
    | tstats avg(count) as avg_count stdev(count) as stdev_count
        where index=firewall earliest=-7d latest=-1h
        by src_prefix64
]

| eval z_score = if(isnotnull(avg_count),
    (current_count - avg_count) / (stdev_count + 1), 99)
| eval status = case(
    isnull(avg_count), "new_source",
    z_score > 3, "anomalous_spike",
    z_score < -3, "anomalous_drop",
    true(), "normal"
)
| where status != "normal"
| table src_prefix64, current_count, avg_count, z_score, status
| sort -z_score
```

## QRadar Building Block Chain

```
# QRadar: multi-event correlation using BB chaining

BB1: IPv6_Scan_Observed
  When: ICMPv6 type 128 OR TCP SYN → drop, from same source, > 50 unique dests in 5m

BB2: IPv6_Auth_Failure
  When: SSH or RDP auth failure from IPv6 source, > 5 in 10m

BB3: IPv6_Auth_Success
  When: SSH or RDP auth success from IPv6 source

# Correlation rule: Scan → Auth Failure → Success
Rule: IPv6_Reconnaissance_Attack_Chain
  Test 1: BB1 fired for source_ip within last 30m
  Test 2: BB2 fired for same source_ip within last 30m
  Test 3: BB3 fires for same source_ip
  Action: Create offense, severity=HIGH
```

## Conclusion

IPv6 SIEM correlation rules are most effective when they track attack chains across multiple log sources and time windows. Key design principles: use /64 prefix as the correlation key (not individual /128 addresses, which change due to privacy extensions), set appropriate time windows (30 minutes for recon-to-exploit), and chain building blocks (scan → auth failure → auth success). Extract and index `source.prefix64` at ingestion time to avoid expensive on-the-fly regex in correlation queries. Baseline deviation detection — comparing current /64 activity to 7-day historical averages using Z-score — catches novel sources that single-event rules miss.
