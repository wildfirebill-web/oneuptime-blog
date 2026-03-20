# How to Create Kibana Dashboards for IPv6 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Kibana, ELK Stack, Dashboards, Network Monitoring

Description: Build Kibana dashboards to visualize IPv6 traffic patterns, create CIDR-based filters, and monitor dual-stack application behavior using Kibana Lens and aggregations.

## Introduction

Kibana's Lens editor and aggregation-based visualizations support IPv6 traffic analysis out of the box when Elasticsearch indices use the `ip` field type. This guide walks through creating dashboards that show top IPv6 sources, traffic trends, and subnet-level breakdowns.

## Prerequisites

- Elasticsearch index with `ip` field type for source/destination fields
- Kibana 8.x connected to the Elasticsearch cluster
- Network log data indexed (see companion post on Elasticsearch IPv6 index configuration)

## Step 1: Create a Data View

In Kibana:
1. Navigate to **Stack Management > Data Views**
2. Create a new data view matching your index pattern: `network-logs-*`
3. Set `@timestamp` as the time field
4. Save and open in **Discover** to verify IPv6 addresses appear in the `client_ip` field

## Step 2: Top IPv6 Sources Visualization (Lens)

1. Open **Visualize Library > Create new visualization > Lens**
2. Select the `network-logs-*` data view
3. Set chart type to **Top values (Bar)**
4. Drag `client_ip` to the **Horizontal axis**
5. Set the metric to **Count of records**
6. In the **Top values** settings, set size to 20
7. Save as "Top IPv6 Source IPs"

## Step 3: IPv6 vs IPv4 Traffic Split (TSVB)

```json
// KQL filter for IPv6-only traffic in Kibana search bar
client_ip: "2001::/3" or client_ip: "fe80::/10" or client_ip: "::1"
```

Create a TSVB time series with two series:
- Series 1: Count with filter `client_ip: *:*` (contains colon = IPv6)
- Series 2: Count with filter `NOT client_ip: *:*` (IPv4)

This gives a stacked area chart showing IPv6 vs IPv4 traffic over time.

## Step 4: CIDR Subnet Filter Panel

```ndjson
// Saved search for RFC 4193 ULA traffic
{
  "query": {
    "term": {
      "client_ip": "fc00::/7"
    }
  }
}
```

```ndjson
// Saved search for global unicast traffic
{
  "query": {
    "term": {
      "client_ip": "2000::/3"
    }
  }
}
```

Add these saved searches as dashboard panels to segment traffic by address category.

## Step 5: Geolocation for IPv6 (if enriched)

If log data includes GeoIP enrichment for IPv6 addresses (via the GeoIP processor):

```json
PUT _ingest/pipeline/geoip-ipv6
{
  "processors": [
    {
      "geoip": {
        "field": "client_ip",
        "target_field": "geoip",
        "database_file": "GeoLite2-City.mmdb"
      }
    }
  ]
}
```

After applying this pipeline, add a **Maps** visualization using `geoip.location` as the geo_point field to plot IPv6 source locations on a world map.

## Step 6: Dashboard Layout

Assemble the following panels into a single dashboard:

| Panel | Type | Purpose |
|-------|------|---------|
| IPv6 vs IPv4 traffic | TSVB time series | Traffic split over time |
| Top 20 IPv6 sources | Lens bar chart | Source IP ranking |
| Response codes by IP | Lens heatmap | Error distribution |
| Global unicast traffic | Saved search | RFC 2000::/3 traffic |
| ULA traffic | Saved search | Internal address traffic |
| Geographic distribution | Maps | Source locations |
| Bytes by source subnet | Lens treemap | Bandwidth by /48 |

## Step 7: Alerting on IPv6 Anomalies

```json
// Kibana Alerting rule: new IPv6 source not seen before (pseudo-config)
POST kbn:/api/alerting/rule
{
  "name": "New IPv6 Source Alert",
  "rule_type_id": ".es-query",
  "consumer": "alerts",
  "schedule": {"interval": "5m"},
  "params": {
    "index": ["network-logs-*"],
    "timeField": "@timestamp",
    "timeWindowSize": 5,
    "timeWindowUnit": "m",
    "size": 100,
    "esQuery": {
      "query": {
        "bool": {
          "filter": [
            {"term": {"client_ip": "2001:db8::/32"}},
            {"term": {"action": "BLOCK"}}
          ]
        }
      }
    },
    "thresholdComparator": ">",
    "threshold": [0]
  }
}
```

## Conclusion

Kibana dashboards for IPv6 traffic leverage Elasticsearch's native `ip` field type to enable subnet filtering with CIDR notation in KQL queries and saved searches. The combination of Lens visualizations for top sources, TSVB for traffic splits, and Maps for geolocation provides comprehensive visibility into IPv6 network activity. Apply GeoIP enrichment at ingest time via ingest pipelines to enable geographic dashboards for IPv6 sources.
