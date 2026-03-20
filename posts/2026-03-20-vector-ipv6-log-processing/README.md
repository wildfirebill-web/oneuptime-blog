# How to Configure Vector for IPv6 Log Processing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Vector, Log Processing, Observability, DevOps

Description: Configure Vector to collect logs over IPv6 interfaces, transform IPv6 address fields using VRL, route by IPv6 subnet, and forward to IPv6-addressed sinks.

## Introduction

Vector is a high-performance observability data pipeline. It supports IPv6 natively - sources can bind to IPv6 addresses and sinks can connect to IPv6 endpoints. Vector's Vector Remap Language (VRL) provides powerful IPv6 address manipulation including parsing, validation, and subnet classification.

## Step 1: Sources on IPv6 Interfaces

```toml
# /etc/vector/vector.toml

# Syslog source bound to all IPv6 interfaces

[sources.syslog_ipv6]
type = "syslog"
mode = "tcp"
address = "[::]:5140"

# HTTP source for log ingestion on IPv6
[sources.http_logs]
type = "http"
address = "[::]:8080"
encoding.codec = "json"

# Socket source (raw TCP) on IPv6
[sources.raw_tcp]
type = "socket"
mode = "tcp"
address = "[::]:5000"
```

## Step 2: Transform IPv6 Fields with VRL

```toml
# Extract and enrich IPv6 addresses
[transforms.enrich_ipv6]
type = "remap"
inputs = ["syslog_ipv6"]
source = '''
  # Parse IPv6 address from the message field
  .client_ip = parse_ip!(.message, format: "regex")

  # Validate and normalize IPv6 addresses
  if is_ip(.client_ip) {
    # Classify address type
    .ip_version = if contains(string!(.client_ip), ":") { "ipv6" } else { "ipv4" }

    # Classify IPv6 category
    if .ip_version == "ipv6" {
      .ipv6_type = if starts_with(string!(.client_ip), "fe80") {
        "link_local"
      } else if starts_with(string!(.client_ip), "::1") {
        "loopback"
      } else if starts_with(string!(.client_ip), "fc") || starts_with(string!(.client_ip), "fd") {
        "ula"
      } else {
        "global_unicast"
      }
    }
  }
'''
```

## Step 3: Route by IPv6 Subnet

```toml
# Route events by IPv6 address type
[transforms.route_by_ip_type]
type = "route"
inputs = ["enrich_ipv6"]

[transforms.route_by_ip_type.route]
ipv6_global = '.ip_version == "ipv6" && .ipv6_type == "global_unicast"'
ipv6_internal = '.ip_version == "ipv6" && (.ipv6_type == "ula" || .ipv6_type == "link_local")'
ipv4 = '.ip_version == "ipv4"'
```

## Step 4: Aggregate IPv6 Metrics

```toml
# Count events per IPv6 source
[transforms.ipv6_aggregation]
type = "aggregate"
inputs = ["route_by_ip_type.ipv6_global"]
interval_ms = 60000

[transforms.ipv6_metrics]
type = "log_to_metric"
inputs = ["route_by_ip_type.ipv6_global"]

[[transforms.ipv6_metrics.metrics]]
type = "counter"
field = "client_ip"
name = "ipv6_requests_total"
[transforms.ipv6_metrics.metrics.tags]
source_ip = "{{client_ip}}"
ip_version = "{{ip_version}}"
```

## Step 5: Sinks to IPv6 Endpoints

```toml
# Send to Elasticsearch over IPv6
[sinks.elasticsearch]
type = "elasticsearch"
inputs = ["route_by_ip_type.ipv6_global", "route_by_ip_type.ipv4"]
endpoint = "http://[2001:db8::10]:9200"
index = "logs-%Y.%m.%d"
auth.strategy = "basic"
auth.user = "elastic"
auth.password = "${ES_PASSWORD}"

# Send to Loki over IPv6
[sinks.loki]
type = "loki"
inputs = ["route_by_ip_type.ipv6_internal"]
endpoint = "http://[2001:db8::20]:3100"
encoding.codec = "json"
labels.job = "vector"
labels.ip_version = "{{ ip_version }}"
labels.ipv6_type = "{{ ipv6_type }}"

# Send to another Vector instance over IPv6
[sinks.vector_aggregator]
type = "vector"
inputs = ["syslog_ipv6"]
address = "[2001:db8::30]:9000"

# Prometheus metrics sink
[sinks.prometheus]
type = "prometheus_exporter"
inputs = ["ipv6_metrics"]
address = "[::]:9598"
```

## Step 6: Full Configuration Example

```toml
# Minimal production Vector config for IPv6 log processing
data_dir = "/var/lib/vector"

[sources.nginx]
type = "file"
include = ["/var/log/nginx/access.log"]
ignore_older_secs = 86400

[transforms.parse_nginx]
type = "remap"
inputs = ["nginx"]
source = '''
  . = parse_nginx_log!(string!(.message), format: "combined")
  .ip_version = if contains(string!(.client), ":") { "ipv6" } else { "ipv4" }
'''

[sinks.out]
type = "elasticsearch"
inputs = ["parse_nginx"]
endpoint = "http://[2001:db8::10]:9200"
index = "nginx-{{ ip_version }}-%Y.%m.%d"
```

## Conclusion

Vector's native IPv6 support spans all pipeline stages: sources bind to `[::]:port` for any IPv6 interface, VRL transforms classify and enrich IPv6 addresses with address type detection, route transforms direct IPv6 and IPv4 traffic to different sinks, and sinks connect to IPv6-addressed backends. VRL's string manipulation functions provide flexible subnet and address type classification without requiring external lookup tables.
