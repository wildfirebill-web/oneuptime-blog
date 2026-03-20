# How to Configure Elastic Stack for IPv6 Log Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elasticsearch, Kibana, Logstash, IPv6, SIEM, ECS, Log Analysis

Description: Configure the Elastic Stack (Elasticsearch, Logstash, Kibana) to ingest, index, and analyze IPv6 network logs using ECS field mappings and grok patterns.

## Elastic Common Schema (ECS) IPv6 Fields

ECS defines standard fields for IPv6 addresses:

| ECS Field | Type | Description |
|---|---|---|
| `source.ip` | ip | Source IPv6 address |
| `destination.ip` | ip | Destination IPv6 address |
| `client.ip` | ip | Client-side IP |
| `server.ip` | ip | Server-side IP |
| `network.type` | keyword | `ipv6` or `ipv4` |
| `source.address` | keyword | Raw address string (with brackets/port) |

Elasticsearch `ip` field type natively stores IPv6 and supports CIDR queries.

## Elasticsearch: Index Mapping for IPv6

```json
PUT /network-logs
{
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "source": {
        "properties": {
          "ip":   { "type": "ip" },
          "port": { "type": "integer" }
        }
      },
      "destination": {
        "properties": {
          "ip":   { "type": "ip" },
          "port": { "type": "integer" }
        }
      },
      "network": {
        "properties": {
          "type":      { "type": "keyword" },
          "protocol":  { "type": "keyword" },
          "direction": { "type": "keyword" }
        }
      },
      "event": {
        "properties": {
          "action":   { "type": "keyword" },
          "outcome":  { "type": "keyword" }
        }
      }
    }
  }
}
```

## Logstash: Grok Patterns for IPv6

```ruby
# /etc/logstash/conf.d/ipv6-network.conf

input {
  beats {
    port => 5044
    host => "::"  # Listen on IPv6
  }
}

filter {
  # Parse firewall logs with IPv6
  if [type] == "firewall" {
    grok {
      match => {
        "message" => "%{SYSLOGTIMESTAMP:timestamp} %{HOSTNAME:hostname} kernel: \[%{NUMBER}\] %{WORD:action} SRC=%{IP:source_ip} DST=%{IP:destination_ip} (?:LEN=%{NUMBER:length} )?.*PROTO=%{WORD:protocol}"
      }
    }

    # Handle IPv6-specific format
    if "_grokparsefailure" in [tags] {
      grok {
        match => {
          "message" => "SRC=(?P<source_ip>[0-9a-fA-F:]+:[0-9a-fA-F:]+) DST=(?P<destination_ip>[0-9a-fA-F:]+:[0-9a-fA-F:]+)"
        }
        tag_on_failure => []
      }
    }

    # ECS field mapping
    mutate {
      rename => { "source_ip"      => "[source][ip]" }
      rename => { "destination_ip" => "[destination][ip]" }
    }

    # Classify address type
    if [source][ip] =~ /^fe80:/ {
      mutate { add_field => { "[source][ip_type]" => "link-local" } }
    } else if [source][ip] =~ /^fc|^fd/ {
      mutate { add_field => { "[source][ip_type]" => "ula" } }
    } else if [source][ip] =~ /[0-9a-fA-F:]+:[0-9a-fA-F:]+/ {
      mutate {
        add_field => { "[source][ip_type]" => "global" }
        add_field => { "[network][type]"   => "ipv6" }
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["[::1]:9200"]  # IPv6 Elasticsearch
    index => "network-logs-%{+YYYY.MM.dd}"
  }
}
```

## Elasticsearch: IPv6 CIDR Queries

```json
// Search for traffic from a specific /48 subnet
GET /network-logs/_search
{
  "query": {
    "term": {
      "source.ip": {
        "value": "2001:db8:corp::/48"
      }
    }
  }
}

// Multiple IPv6 subnets
GET /network-logs/_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "source.ip": "2001:db8:corp::/48" } },
        { "term": { "source.ip": "2001:db8:dmz::/48" } }
      ]
    }
  }
}

// Find ULA sources contacting external destinations
GET /network-logs/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "source.ip": "fc00::/7" } }
      ],
      "must_not": [
        { "term": { "destination.ip": "fc00::/7" } },
        { "term": { "destination.ip": "fe80::/10" } }
      ]
    }
  }
}
```

## Elasticsearch: IPv6 Aggregations

```json
// Top source IPv6 /64 prefixes by event count
GET /network-logs/_search
{
  "aggs": {
    "src_prefix64": {
      "ip_prefix": {
        "field": "source.ip",
        "prefix_length": 64
      }
    }
  }
}

// Count events by IPv6 address type
GET /network-logs/_search
{
  "aggs": {
    "by_type": {
      "terms": {
        "field": "source.ip_type"
      }
    }
  }
}
```

## Kibana: IPv6 Visualization

```json
// Kibana saved search: IPv6 traffic anomalies
// Discover filter: network.type is ipv6

// KQL queries in Kibana:
// Traffic from specific subnet:
source.ip: "2001:db8::/32"

// Find link-local sources (should not be in forwarded logs):
source.ip: "fe80::/10"

// Find IPv4-mapped addresses (dual-stack indicator):
source.ip: "::ffff:0:0/96"

// High port destinations from IPv6:
network.type: ipv6 AND destination.port > 1024 AND event.action: drop
```

## Filebeat: IPv6 Module Configuration

```yaml
# /etc/filebeat/filebeat.yml

filebeat.inputs:
  - type: log
    paths:
      - /var/log/ip6tables.log
    fields:
      type: firewall
      network_version: ipv6

output.logstash:
  hosts:
    - "[2001:db8::logstash]:5044"  # Connect to Logstash via IPv6

# Enable iptables module (handles IPv6 logs)
filebeat.modules:
  - module: iptables
    log:
      enabled: true
      var.paths: ["/var/log/ip6tables.log"]
```

## Conclusion

Elastic Stack handles IPv6 natively through the `ip` field type in Elasticsearch, which accepts both IPv4 and IPv6 and supports CIDR prefix queries. Use ECS field names (`source.ip`, `destination.ip`) for compatibility with built-in Kibana security dashboards. In Logstash, use grok with the `%{IP}` pattern (which matches IPv6) or custom regex for non-standard formats. Logstash and Elasticsearch can listen on IPv6 by setting `host: "::"`. Use `ip_prefix` aggregation to summarize traffic by /64 prefix - essential for analyzing large IPv6 address spaces where individual /128 addresses are less meaningful than /64 blocks.
