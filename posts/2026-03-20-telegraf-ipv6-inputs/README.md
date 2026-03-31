# How to Configure Telegraf with IPv6 Inputs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Telegraf, IPv6, InfluxDB, Monitoring, Metric, Time Series

Description: A guide to configuring Telegraf input plugins to collect IPv6 metrics, scrape IPv6 endpoints, and report data to InfluxDB or other outputs.

Telegraf is InfluxData's agent for collecting and reporting metrics. It supports IPv6 across its input and output plugins, allowing it to scrape IPv6 endpoints and write to IPv6-addressed InfluxDB instances.

## Step 1: Configure Telegraf to Listen on IPv6

```toml
# telegraf.conf - Core agent configuration

[agent]
  interval = "10s"
  hostname = ""  # Auto-detect hostname

# Output to InfluxDB (supports IPv6 endpoints)
[[outputs.influxdb_v2]]
  urls = ["http://[2001:db8::influx]:8086"]
  token = "$INFLUX_TOKEN"
  organization = "my-org"
  bucket = "ipv6-metrics"
```

## Step 2: Collect Linux IPv6 Network Stats

```toml
# net input plugin - Collects per-interface statistics
# IPv6 traffic appears in standard interface metrics
[[inputs.net]]
  # Specify interfaces to collect (or leave empty for all)
  interfaces = ["eth0", "eth1"]

  # This collects bytes sent/received per interface
  # For dedicated IPv6 stats, use the kernel input

[[inputs.kernel]]
  # Collects /proc/net/snmp6 statistics
  # Provides dedicated IPv6 packet counters

# Optionally use the netstat input for connection state
[[inputs.netstat]]
  # Reports per-protocol connection counts including tcp6, udp6
```

## Step 3: Scrape Prometheus IPv6 Targets with Telegraf

```toml
# prometheus input - Scrape Prometheus metrics from IPv6 endpoints
[[inputs.prometheus]]
  # List of IPv6 Prometheus endpoints to scrape
  urls = [
    "http://[2001:db8::10]:9100/metrics",  # Node Exporter IPv6
    "http://[2001:db8::11]:9100/metrics",
    "http://[::1]:9090/metrics"             # Local Prometheus
  ]

  # Namespace to add to metric names
  metric_version = 2

  # Tags to add to all metrics from these targets
  [inputs.prometheus.tags]
    environment = "production"
    ip_family = "ipv6"
```

## Step 4: HTTP Check Plugin for IPv6 Endpoints

```toml
# http_response input - Active check for IPv6 HTTP endpoints
[[inputs.http_response]]
  address = "http://[2001:db8::10]:80/"
  method = "GET"
  response_timeout = "10s"
  follow_redirects = true

  # Tag the metric with ip_family
  [inputs.http_response.tags]
    instance = "web-01-ipv6"
    ip_family = "ipv6"

[[inputs.http_response]]
  address = "https://www.example.com/"
  method = "GET"
  response_timeout = "10s"

  [inputs.http_response.tags]
    instance = "example-com-ipv6"
```

## Step 5: Ping Plugin for IPv6 ICMP Checks

```toml
# ping input - ICMP ping to IPv6 addresses
[[inputs.ping]]
  urls = [
    "2001:db8::10",
    "2001:db8::11",
    "2001:4860:4860::8888"  # Google IPv6 DNS
  ]

  # Number of pings
  count = 3

  # Ping interval in seconds
  ping_interval = 1.0

  # Timeout per ping
  deadline = 10

  # Force IPv6 (-6 flag)
  method = "native"
  ipv6 = true
```

## Step 6: DNS Input for IPv6 AAAA Record Monitoring

```toml
# dns_query input - Check IPv6 DNS record resolution
[[inputs.dns_query]]
  # DNS servers to query (IPv6 DNS resolvers)
  servers = ["2001:4860:4860::8888", "2606:4700:4700::1111"]

  # Domain to query
  domains = ["example.com", "www.example.com"]

  # Query type AAAA for IPv6 records
  record_type = "AAAA"

  # Timeout
  timeout = 5
```

## Step 7: Test the Configuration

```bash
# Test Telegraf config syntax
telegraf --config /etc/telegraf/telegraf.conf --test

# Run in debug mode to see collected metrics
telegraf --config /etc/telegraf/telegraf.conf --debug 2>&1 | head -50

# Check specific plugin
telegraf --config /etc/telegraf/telegraf.conf --input-filter ping --test
```

## Key IPv6 Metrics from Telegraf

```text
# From net plugin
net_bytes_recv{interface="eth0"} = X        # Total bytes received
net_bytes_sent{interface="eth0"} = X

# From http_response plugin
http_response_response_time{server="[2001:db8::10]:80"} = 0.025

# From ping plugin
ping_average_response_ms{url="2001:4860:4860::8888"} = 12.3
ping_packet_loss_percent{url="2001:4860:4860::8888"} = 0.0
```

Telegraf's broad plugin ecosystem makes it an excellent single-agent solution for collecting IPv6 metrics from network interfaces, HTTP endpoints, DNS resolvers, and ICMP targets in one unified configuration.
