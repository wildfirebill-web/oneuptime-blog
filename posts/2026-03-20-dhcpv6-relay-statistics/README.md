# How to Monitor DHCPv6 Relay Statistics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, Relay, Statistics, Monitoring, Prometheus, Networking

Description: Monitor DHCPv6 relay statistics across platforms including message counters, error rates, and Prometheus metrics for observability.

## Statistics Available on Each Platform

| Platform | Command | Key Metrics |
|---|---|---|
| Linux dhcrelay | Via syslog | SOLICIT, REQUEST counts |
| Cisco IOS | show ipv6 dhcp relay statistics | Bindings, drops |
| Juniper | show dhcp v6 relay statistics | Messages, errors |
| ISC Kea | REST API | Per-subnet counters |
| Windows Server | netsh dhcp show statistics | Leases, messages |

## Cisco IOS/IOS-XE Statistics

```
! Show relay message statistics
show ipv6 dhcp relay statistics

! Sample output:
! Packets relayed:
!   Received:        1523   Total:       1523
!   Forwarded:       1519   Dropped:        4
!   SOLICIT:          387   ADVERTISE:      387
!   REQUEST:          387   REPLY:          387
!   RENEW:            123   RELEASE:         89

! Reset counters
clear ipv6 dhcp relay statistics

! Show per-interface
show ipv6 dhcp interface GigabitEthernet0/1
```

## Juniper Relay Statistics

```
# Show DHCPv6 relay statistics
show dhcp v6 relay statistics

# Output fields:
# Messages sent to server:    1200
# Messages sent to client:    1198
# Messages dropped:               2
# SOLICIT:                      400
# ADVERTISE:                    400
# REQUEST:                      400
# REPLY:                        398
# RENEW:                        100
# RELEASE:                       56

# Clear statistics
clear dhcp v6 relay statistics all

# Per-group statistics
show dhcp v6 relay statistics group CLIENTS
```

## ISC Kea Statistics via REST API

```python
#!/usr/bin/env python3
# kea-relay-stats.py — Query Kea DHCP6 statistics via REST API

import requests
import json

KEA_API = "http://[2001:db8::dhcp-server]:8000"

def get_kea_stats():
    resp = requests.post(
        f"{KEA_API}/",
        json={
            "command": "statistic-get-all",
            "service": ["dhcp6"],
            "arguments": {}
        }
    )
    data = resp.json()

    stats = data[0].get("arguments", {})

    # DHCPv6 relay-relevant statistics
    relay_stats = {
        "pkt6-received": stats.get("pkt6-received", [[0]])[0][0],
        "pkt6-solicit-received": stats.get("pkt6-solicit-received", [[0]])[0][0],
        "pkt6-request-received": stats.get("pkt6-request-received", [[0]])[0][0],
        "pkt6-reply-sent": stats.get("pkt6-reply-sent", [[0]])[0][0],
        "pkt6-advertise-sent": stats.get("pkt6-advertise-sent", [[0]])[0][0],
    }

    return relay_stats

if __name__ == "__main__":
    stats = get_kea_stats()
    for key, value in stats.items():
        print(f"{key}: {value}")
```

## Prometheus Metrics for DHCPv6 Relay

```python
#!/usr/bin/env python3
# dhcpv6-relay-exporter.py — Prometheus exporter for DHCPv6 relay stats

from prometheus_client import start_http_server, Gauge, Counter
import subprocess
import time
import re

# Metrics
relay_received = Counter('dhcpv6_relay_received_total', 'Messages received from clients', ['message_type'])
relay_forwarded = Counter('dhcpv6_relay_forwarded_total', 'Messages forwarded to server')
relay_dropped = Counter('dhcpv6_relay_dropped_total', 'Messages dropped')

def collect_dhcrelay_stats():
    """Parse dhcrelay syslog for statistics"""
    # In production, use journalctl to parse dhcrelay output
    result = subprocess.run(
        ['journalctl', '-u', 'isc-dhcp-relay6', '--since', '1 minute ago', '--no-pager'],
        capture_output=True, text=True
    )

    solicit_count = len(re.findall(r'RELAY-FORW.*SOLICIT', result.stdout))
    request_count = len(re.findall(r'RELAY-FORW.*REQUEST', result.stdout))
    drop_count = len(re.findall(r'drop', result.stdout, re.IGNORECASE))

    if solicit_count > 0:
        relay_received.labels(message_type='solicit').inc(solicit_count)
    if request_count > 0:
        relay_received.labels(message_type='request').inc(request_count)
    if drop_count > 0:
        relay_dropped.inc(drop_count)

def main():
    start_http_server(9200)
    print("DHCPv6 relay exporter on :9200/metrics")

    while True:
        collect_dhcrelay_stats()
        time.sleep(60)

if __name__ == '__main__':
    main()
```

## Grafana Dashboard Queries

```
# Prometheus queries for DHCPv6 relay monitoring

# Message rate per type
rate(dhcpv6_relay_received_total[5m])

# Drop rate (alert if > 1%)
rate(dhcpv6_relay_dropped_total[5m]) /
rate(dhcpv6_relay_received_total[5m]) * 100

# Active DHCP bindings (from Kea)
kea_dhcp6_subnet_assigned_addresses

# Relay-forward to relay-reply ratio (should be ~1)
rate(pkt6_relay_forw_sent_total[5m]) /
rate(pkt6_relay_repl_received_total[5m])
```

## Alerting on Relay Issues

```yaml
# Prometheus alerting rules
groups:
  - name: dhcpv6-relay
    rules:
      - alert: DHCPv6RelayDropsHigh
        expr: rate(dhcpv6_relay_dropped_total[5m]) > 10
        for: 2m
        annotations:
          summary: "DHCPv6 relay dropping {{ $value }} msgs/s"

      - alert: DHCPv6RelayNoTraffic
        expr: rate(dhcpv6_relay_received_total[15m]) == 0
        for: 10m
        annotations:
          summary: "DHCPv6 relay receiving no traffic — clients may not be getting addresses"
```

## Conclusion

DHCPv6 relay statistics reveal address assignment health at a glance. Cisco and Juniper provide per-interface relay counters via CLI. ISC Kea exposes rich per-subnet statistics via REST API. Export these metrics to Prometheus for time-series monitoring and alerting. Key alert conditions: high drop rates (> 1%), no traffic (relay daemon stopped), and RELAY-FORW without RELAY-REPL (server unreachable). A healthy relay shows balanced SOLICIT/ADVERTISE/REQUEST/REPLY ratios.
