# How to Write IPv6 Threat Detection Rules in SIEM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, SIEM, Threat Detection, Security Rules, Sigma, Detection Engineering

Description: Write effective threat detection rules for IPv6-specific attacks and anomalies using Sigma rule format and SIEM platform translations for Splunk, Elastic, and QRadar.

## IPv6-Specific Threat Categories

| Threat | Description | Detection Method |
|---|---|---|
| NDP cache overflow | Flood INCOMPLETE entries to exhaust router cache | Rate: NS messages > threshold |
| Router Advertisement spoofing | Fake RA announcements to redirect traffic | Unauthorized RA source |
| 6to4/Teredo tunneling | Bypass firewall via IPv6 tunnels | Protocol 41, UDP 3544 |
| IPv6 scanning | /64 prefix scanning to find active hosts | High ICMP6/SYN to unique /128s |
| DHCPv6 starvation | Exhaust DHCPv6 address pool | Rate: Solicit from unique MACs |
| IPv6 header manipulation | Extension header abuse | Unusual header chains |

## Sigma Rule Format for IPv6 Threats

```yaml
# sigma-ndp-flood.yml

# Detect NDP cache overflow attack

title: NDP Cache Overflow Attack
id: a1b2c3d4-e5f6-7890-abcd-ef1234567890
status: stable
description: Detects high rate of ICMPv6 Neighbor Solicitation messages
  indicating a possible NDP cache exhaustion attack
author: Security Team
date: 2026-03-20
tags:
    - attack.impact
    - attack.t1499  # Endpoint Denial of Service
logsource:
    category: network
    product: firewall
detection:
    selection:
        ip_version: ipv6
        protocol: icmpv6
        icmpv6_type: 135  # Neighbor Solicitation
    timeframe: 60s
    condition: selection | count() by dst_ip > 500
falsepositives:
    - Large network with many hosts restarting simultaneously
level: high
```

```yaml
# sigma-rogue-ra.yml
# Detect Rogue Router Advertisement

title: Rogue IPv6 Router Advertisement
id: b2c3d4e5-f6a7-8901-bcde-f12345678901
status: stable
description: Detects ICMPv6 Router Advertisement from unauthorized source
author: Security Team
date: 2026-03-20
tags:
    - attack.lateral_movement
    - attack.t1557.001  # LLMNR/NBT-NS Poisoning (closest)
logsource:
    category: network
    product: firewall
detection:
    selection:
        protocol: icmpv6
        icmpv6_type: 134  # Router Advertisement
    filter_legitimate:
        src_ip|cidr:
            - '2001:db8:infra::/48'  # Known router subnets
    condition: selection and not filter_legitimate
falsepositives:
    - New router deployment
    - Misconfigured radvd
level: high
```

## Splunk: IPv6 Threat Detection SPL

```text
| SPL: Detect IPv6 scanning activity

index=firewall protocol=tcp action=drop
| where cidrmatch("::/0", src_ip) AND NOT cidrmatch("::ffff:0:0/96", src_ip)
| bin _time span=5m
| stats dc(dst_ip) as unique_dests, count as packets by src_ip, _time
| where unique_dests > 50
| eval threat="IPv6_port_scan"
| table _time, src_ip, unique_dests, packets, threat
| sort -unique_dests
```

```text
| SPL: Detect NDP flooding

index=firewall protocol=icmpv6 icmpv6_type=135
| bin _time span=1m
| stats count as ns_count, dc(src_ip) as sources by dst_ip, _time
| where ns_count > 300
| eval severity=if(ns_count > 1000, "critical", "high")
| table _time, dst_ip, ns_count, sources, severity
```

```text
| SPL: Detect 6to4 and Teredo tunneling

index=network
| where (proto=41 OR (proto=17 AND dst_port=3544))
| stats count by src_ip, dst_ip, proto
| eval tunnel_type=if(proto=41, "6to4", "Teredo")
| table src_ip, dst_ip, tunnel_type, count
| sort -count
```

## Elastic: Detection Rule YAML

```yaml
# Elastic Security detection rule for IPv6 scan
name: IPv6 Port Scan Detected
description: Detects potential IPv6 port scanning activity
risk_score: 73
severity: high
type: threshold
threshold:
  field: destination.ip
  value: 50
query: >
  network.type: ipv6
  AND event.action: (deny OR drop OR blocked)
  AND NOT source.ip: (fc00::/7 OR fe80::/10)
from: now-5m
interval: 1m

---
# Rogue RA rule
name: Rogue IPv6 Router Advertisement
description: ICMPv6 RA from non-router source
type: eql
language: eql
query: |
  network where
    network.type == "ipv6" and
    network.protocol == "icmpv6" and
    // ICMPv6 type 134 = Router Advertisement
    event.dataset == "firewall" and
    not CIDR_MATCH("2001:db8:infra::/48", source.ip) and
    not CIDR_MATCH("fe80::/10", source.ip)
```

## Sigma-to-Platform Conversion

```bash
# Install sigma CLI tool
pip install sigma-cli
pip install pysigma-backend-splunk
pip install pysigma-backend-elasticsearch
pip install pysigma-backend-qradar

# Convert Sigma rule to Splunk SPL
sigma convert --target splunk --pipeline ecs_windows \
    sigma-ndp-flood.yml

# Convert to Elasticsearch EQL
sigma convert --target elasticsearch --pipeline ecs_windows \
    sigma-rogue-ra.yml

# Convert to QRadar AQL
sigma convert --target qradar \
    sigma-ndp-flood.yml
```

## Tuning Rules to Reduce False Positives

```yaml
# Base rule with tuning parameters
title: IPv6 Internal Scanning
detection:
    selection:
        src_ip|cidr: '::/0'
        dst_ip|cidr: '2001:db8:internal::/48'
        event.action: 'deny'
    filter_internal_scanners:
        # Exclude known security scanners
        src_ip|cidr:
            - '2001:db8:security:scanner::/64'
    filter_backup_systems:
        # Exclude backup traffic patterns
        dst_port:
            - 445
            - 139
        src_ip|cidr: '2001:db8:backup::/48'
    condition: selection and not (filter_internal_scanners or filter_backup_systems)
    timeframe: 5m
    count(dst_ip) > 20 by src_ip
```

## Conclusion

Effective IPv6 threat detection requires understanding IPv6-specific attack vectors. Use Sigma format for platform-independent rule authoring and convert to Splunk SPL, Elastic EQL, or QRadar AQL. The most critical rules to deploy: NDP flood detection (NS rate per target > 300/min), rogue RA detection (RA from non-router sources), and IPv6 scanning (> 50 unique /128 destinations from one /64 in 5 minutes). Filter noise by whitelisting known infrastructure, security scanners, and backup systems. Always include `not filter_legitimate_routers` in RA rules to avoid alerting on actual router restarts.
