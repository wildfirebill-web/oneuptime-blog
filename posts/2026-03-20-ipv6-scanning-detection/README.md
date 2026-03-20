# How to Detect IPv6 Network Scanning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Network Scanning, Security Detection, SIEM, Threat Detection, Suricata

Description: Detect IPv6 network scanning activity including /64 prefix scanning, ICMPv6 probes, and port scans using firewall logs, IDS signatures, and SIEM correlation.

## IPv6 Scanning Characteristics

IPv6 scanning differs fundamentally from IPv4 scanning:

| Aspect | IPv4 | IPv6 |
|---|---|---|
| Subnet size | /24 = 256 hosts | /64 = 18 quintillion |
| Sequential scan | Common | Impractical |
| Multicast probing | Rare | All-nodes (ff02::1) is quick |
| Hitlist scanning | Less common | Primary method |
| DNS enumeration | Secondary | Primary reconnaissance |

Attackers targeting IPv6 use: all-nodes multicast ping, DNS zone transfer, search engines (Shodan), and addresses from leaked IPv6 logs.

## Detection Method 1: Firewall Drop Rate

```text
# High rate of drops to unique /128 destinations = scanning

# Splunk: detect IPv6 port scanning

index=firewall action=drop network_type=ipv6
| bin _time span=5m
| stats
    dc(dst_ip) as unique_dests,
    dc(dst_port) as unique_ports,
    count as total_drops
    by src_ip, _time
| where unique_dests > 30 OR unique_ports > 20
| eval scan_type=case(
    unique_dests > 30 AND unique_ports < 5, "host_scan",
    unique_ports > 20 AND unique_dests < 5, "port_scan",
    unique_dests > 10 AND unique_ports > 10, "combined_scan"
)
| table _time, src_ip, unique_dests, unique_ports, total_drops, scan_type
```

## Detection Method 2: ICMPv6 Echo Probing

```bash
# Suricata: detect ICMPv6 echo flood to multiple destinations
# /etc/suricata/rules/ipv6-scan.rules

# Detect ICMPv6 echo to all-nodes multicast
alert icmp6 any any -> ff02::1 any (
    msg:"IPv6 All-Nodes Multicast Ping - Reconnaissance";
    itype:128;
    threshold: type both, track by_src, count 5, seconds 10;
    sid:9001001; rev:1;
    classtype:network-scan;
)

# Detect high-rate ICMPv6 to unique /64 prefix
alert icmp6 $EXTERNAL_NET any -> $HOME_NET any (
    msg:"IPv6 Prefix Scanning via ICMPv6 Echo";
    itype:128;
    threshold: type threshold, track by_src, count 50, seconds 60;
    sid:9001002; rev:1;
    classtype:network-scan;
)

# Detect TCP SYN flood to IPv6 hosts (port scan)
alert tcp $EXTERNAL_NET any -> $HOME_NET any (
    msg:"IPv6 TCP Port Scan";
    flags:S,12;
    threshold: type threshold, track by_src, count 30, seconds 30;
    sid:9001003; rev:1;
    classtype:network-scan;
)
```

## Detection Method 3: DNS Reconnaissance

```text
# Attackers enumerate IPv6 by DNS zone transfer or AAAA queries

# Splunk: detect DNS enumeration for IPv6 hosts
index=dns query_type=AAAA
| bin _time span=1m
| stats
    dc(query_name) as unique_queries,
    count as total_queries
    by src_ip, _time
| where unique_queries > 100
| eval threat="DNS_IPv6_Enumeration"
| table _time, src_ip, unique_queries, total_queries, threat

# Detect DNS zone transfer attempts
index=dns query_type=AXFR
| stats count by src_ip, query_name
| where count > 0
| eval threat="DNS_Zone_Transfer_Attempt"
```

## Detection Method 4: NDP-Based Host Discovery

```bash
# Attackers send NS to solicited-node multicast for each /64 address
# This generates INCOMPLETE entries and shows in NDP table

# Monitor NDP table growth rate
#!/bin/bash
PREV_COUNT=0
while true; do
    CURRENT=$(ip -6 neigh show nud incomplete | wc -l)
    DELTA=$((CURRENT - PREV_COUNT))

    if [ ${DELTA} -gt 50 ]; then
        echo "$(date): ALERT: ${DELTA} new INCOMPLETE NDP entries in 30s"
        echo "Total INCOMPLETE: ${CURRENT}"

        # Log top sources from NDP scanning
        ip -6 neigh show nud incomplete | \
            awk '{print $3}' | sort | uniq -c | sort -rn | head -5
    fi

    PREV_COUNT=${CURRENT}
    sleep 30
done
```

## Sigma Rule: IPv6 Scanning

```yaml
title: IPv6 Host Discovery Scan
id: c3d4e5f6-a7b8-9012-cdef-123456789012
status: stable
description: Detects potential IPv6 host discovery via multicast ping or high-rate unicast probes
author: Security Team
date: 2026-03-20
tags:
    - attack.reconnaissance
    - attack.t1595.001
logsource:
    category: network
    product: firewall
detection:
    multicast_probe:
        dst_ip: 'ff02::1'
        protocol: icmpv6
        icmpv6_type: 128
    unicast_scan:
        ip_version: ipv6
        event.action: 'deny'
    condition: multicast_probe or unicast_scan
    timeframe: 1m
    count(dst_ip) by src_ip > 30  # For unicast_scan
falsepositives:
    - Network management tools (Nagios, SNMP discovery)
    - IPv6 reachability testing
level: medium
```

## Automated Blocking Response

```bash
#!/bin/bash
# auto-block-ipv6-scanner.sh - Block detected scanners via ip6tables

THRESHOLD=50     # unique destinations in 5 minutes
CHECK_INTERVAL=60  # seconds

# Requires: iptstate, ss, or firewall log parsing
while true; do
    # Get top IPv6 sources with drops in last 5m
    # (Using log file - replace with live firewall query)
    SCANNERS=$(grep -E "$(date '+%b %e')" /var/log/ip6tables.log | \
        grep " DROP " | \
        awk '{print $11}' | sed 's/SRC=//' | \
        sort | uniq -c | sort -rn | \
        awk -v threshold="${THRESHOLD}" '$1 > threshold {print $2}')

    for SCANNER in ${SCANNERS}; do
        # Check if already blocked
        if ! ip6tables -L INPUT -n | grep -q "${SCANNER}"; then
            echo "$(date): Blocking IPv6 scanner: ${SCANNER}"
            ip6tables -I INPUT -s "${SCANNER}" -j DROP
            ip6tables -I FORWARD -s "${SCANNER}" -j DROP

            # Auto-unblock after 1 hour
            (sleep 3600 && \
             ip6tables -D INPUT -s "${SCANNER}" -j DROP && \
             ip6tables -D FORWARD -s "${SCANNER}" -j DROP) &
        fi
    done

    sleep ${CHECK_INTERVAL}
done
```

## Conclusion

IPv6 scanning detection requires different thresholds than IPv4 due to the vast address space. Key detection signals: drops to > 30 unique /128 destinations from one source in 5 minutes (host scan), > 20 unique ports to one destination (port scan), ICMPv6 echo to ff02::1 (multicast reconnaissance), DNS AAAA bulk queries > 100/minute (DNS enumeration). Suricata rules using `threshold: type threshold, track by_src` provide efficient kernel-level detection. Correlate NDP INCOMPLETE entry growth rate with SIEM events to detect NDP-based host discovery. Use /64 prefix grouping for attribution - IPv6 scanner may rotate between /128 addresses within a /64.
