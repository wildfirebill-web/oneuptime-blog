# How to Build IPv6 Security Dashboards in SIEM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, SIEM, Dashboards, Security Monitoring, Grafana, Kibana, Splunk

Description: Design and build effective IPv6 security dashboards in Splunk, Kibana, and Grafana to visualize traffic patterns, threats, and protocol health at a glance.

## Dashboard Design Principles for IPv6

IPv6 dashboards need to account for:
- **Address space size**: /64 prefixes are more meaningful than individual /128s
- **Address types**: global, ULA, link-local, multicast each have distinct behaviors
- **Protocol specifics**: NDP, RA, DHCPv6 have no IPv4 equivalent
- **Dual-stack coexistence**: show IPv4 vs IPv6 split in traffic

## Panel 1: IPv6 vs IPv4 Traffic Split

```text
# Splunk: traffic version split (timechart)

index=firewall
| eval ip_version=if(match(src_ip, ":"), "IPv6", "IPv4")
| timechart span=1h count by ip_version

# Grafana/Prometheus query:
# sum(rate(firewall_packets_total[5m])) by (ip_version)
```

## Panel 2: IPv6 Address Type Distribution

```text
# Splunk: classify source address types
index=firewall src_ip="*:*"
| eval addr_type=case(
    match(src_ip, "^::1$"), "loopback",
    match(src_ip, "^fe80:"), "link-local",
    match(src_ip, "^fc|^fd"), "ula",
    match(src_ip, "^ff"), "multicast",
    match(src_ip, "^2002:"), "6to4",
    match(src_ip, "^::ffff:"), "ipv4-mapped",
    true(), "global"
)
| stats count by addr_type
| sort -count

# Visualization: pie chart or donut chart
# Expected healthy values:
# global: ~90%, link-local: ~5%, ULA: ~3-5%, other: <1%
```

## Panel 3: Top IPv6 /64 Source Prefixes

```text
# Splunk: aggregate by /64 (more meaningful than /128)
index=firewall src_ip="*:*"
| rex field=src_ip "^(?P<prefix64>(?:[0-9a-fA-F:]{0,4}:){4})"
| stats count as events, dc(dst_ip) as unique_dests by prefix64
| sort -events
| head 20

# Kibana Vega: IPv6 prefix treemap
# Elastic aggregation: ip_prefix on source.ip with prefix_length=64
{
  "aggs": {
    "top_prefixes": {
      "ip_prefix": {
        "field": "source.ip",
        "prefix_length": 64
      },
      "aggs": {
        "event_count": { "value_count": { "field": "_id" } }
      }
    }
  }
}
```

## Panel 4: NDP Protocol Health

```text
# Splunk: NDP message type breakdown
index=firewall protocol=icmpv6
| eval ndp_type=case(
    icmpv6_type=133, "Router Solicitation",
    icmpv6_type=134, "Router Advertisement",
    icmpv6_type=135, "Neighbor Solicitation",
    icmpv6_type=136, "Neighbor Advertisement",
    icmpv6_type=137, "Redirect",
    true(), "Other ICMPv6"
)
| timechart span=5m count by ndp_type

# Alert thresholds to annotate:
# NS > 500/min = potential flood
# RA from unexpected source = rogue RA
```

## Panel 5: IPv6 Security Events

```text
# Splunk: security events over time, categorized
index=security OR index=ids
| eval event_category=case(
    match(description, "(?i)scan"), "Scanning",
    match(description, "(?i)rogue.*RA|RA.*spoof"), "Rogue RA",
    match(description, "(?i)NDP.*flood|NS.*flood"), "NDP Flood",
    match(description, "(?i)poisoning"), "Cache Poisoning",
    match(description, "(?i)exfil"), "Data Exfiltration",
    true(), "Other"
)
| timechart span=1h count by event_category

# Color coding:
# Scanning: yellow (medium severity)
# Rogue RA: red (critical)
# NDP Flood: red (critical)
# Cache Poisoning: red (critical)
```

## Panel 6: Denied IPv6 Connections by Country

```text
# Splunk: geo-lookup for IPv6 sources
index=firewall action=drop src_ip="*:*"
| iplocation src_ip
| stats count by Country
| geom geo_countries featureIdField="Country"

# Note: IPv6 geolocation databases may have lower accuracy than IPv4
# Use MaxMind GeoIP2 for best IPv6 coverage
```

## Grafana Dashboard: IPv6 Infrastructure Metrics

```yaml
# grafana-dashboard.yml structure
# Combine network metrics with security events

panels:
  - title: "IPv6 Traffic Rate"
    type: timeseries
    targets:
      - expr: 'sum(rate(network_bytes_total{ip_version="ipv6"}[5m])) by (interface)'

  - title: "NDP Cache Size"
    type: gauge
    targets:
      - expr: 'ndp_cache_total{iface="eth0"}'
    thresholds:
      - value: 5000
        color: yellow
      - value: 8000
        color: red

  - title: "Active IPv6 RADIUS Sessions"
    type: stat
    targets:
      - expr: 'radius_ipv6_sessions'

  - title: "NDP Message Rate"
    type: timeseries
    targets:
      - expr: 'rate(ndp_messages_total{type="neighbor_solicitation"}[1m])'
      - expr: 'rate(ndp_messages_total{type="router_advertisement"}[1m])'

  - title: "Security Alerts by Type"
    type: bargauge
    targets:
      - expr: 'sum(increase(security_alerts_total{ip_version="ipv6"}[1h])) by (alert_type)'
```

## Kibana: IPv6 Security Overview Dashboard

```json
{
  "title": "IPv6 Security Overview",
  "panels": [
    {
      "type": "metric",
      "title": "IPv6 Events (24h)",
      "query": "network.type: ipv6"
    },
    {
      "type": "area",
      "title": "IPv6 Traffic by Action",
      "query": "network.type: ipv6",
      "split_by": "event.action"
    },
    {
      "type": "data_table",
      "title": "Top Denied IPv6 Sources",
      "query": "network.type: ipv6 AND event.action: deny",
      "columns": ["source.ip", "source.prefix64", "count"]
    },
    {
      "type": "map",
      "title": "IPv6 Source Geography",
      "query": "network.type: ipv6",
      "field": "source.geo.country_iso_code"
    }
  ]
}
```

## Conclusion

Effective IPv6 security dashboards require IPv6-aware panel design. Use /64 prefix aggregation for top-source panels - individual /128 addresses are ephemeral due to privacy extensions. Always show the IPv4/IPv6 traffic split to track dual-stack adoption. Include NDP health panels (NS/NA/RA rates) alongside traditional connection metrics - NDP anomalies are IPv6-specific attack signals that have no IPv4 equivalent. In Grafana, define thresholds on NDP cache size (> 80% of gc_thresh3 = yellow, > 90% = red). In Kibana, use the `ip_prefix` aggregation at /64 for geographic and volume analysis.
