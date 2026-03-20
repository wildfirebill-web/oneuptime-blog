# How to Use Loki LogQL to Query Logs by IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Loki, LogQL, IPv4, Log Analysis, Grafana, Observability

Description: Query Grafana Loki logs by IPv4 address using LogQL, including label filtering, line filters, structured metadata extraction, and metric queries for rate analysis.

## Introduction

Grafana Loki stores logs indexed by labels. LogQL provides log stream selectors and filter expressions for querying by IPv4 address, whether stored as a label or embedded in log lines.

## Basic Line Filter (IP Anywhere in Log Line)

```logql
# Find all logs containing a specific IP
{job="nginx"} |= "203.0.113.42"

# Regex match for IP
{job="nginx"} |~ "203\\.0\\.113\\.[0-9]+"

# Exclude log lines from internal IPs
{job="nginx"} != "10.1." != "192.168."
```

## Parse IP as a Label with JSON or Pattern

```logql
# If logs are JSON with a "client_ip" field
{job="app"} | json | line_format "{{.client_ip}} {{.status}}"

# Pattern parser for custom log format
{job="nginx"} | pattern `<ip> - - [<_>] "<method> <path> <_>" <status> <_>`
              | ip = "203.0.113.42"

# Unpack common log format
{job="nginx"} | regexp `^(?P<client_ip>[^ ]+) - - \[(?P<time>[^\]]+)\] "(?P<method>[^ ]+) (?P<path>[^ ]+)[^"]*" (?P<status>[0-9]+)`
```

## Filter by Parsed IP Label

```logql
# After extracting client_ip with regexp or json parser
{job="nginx"}
  | regexp `^(?P<client_ip>\S+)`
  | client_ip = "203.0.113.42"

# Multiple IPs
{job="nginx"}
  | regexp `^(?P<client_ip>\S+)`
  | client_ip =~ "203\\.0\\.113\\..*|198\\.51\\.100\\..*"
```

## Metric Queries — Request Rate by IP

```logql
# Request rate per minute from each IP
sum by (client_ip) (
  rate(
    {job="nginx"}
      | regexp `^(?P<client_ip>\S+)`
      [1m]
  )
)

# Top 10 IPs by request rate
topk(10,
  sum by (client_ip) (
    rate({job="nginx"} | regexp `^(?P<client_ip>\S+)` [5m])
  )
)
```

## Count 4xx Errors by IP

```logql
sum by (client_ip) (
  count_over_time(
    {job="nginx"}
      | regexp `^(?P<client_ip>\S+) [^ ]+ [^ ]+ \[[^\]]+\] "[^"]*" (?P<status>[45][0-9]{2})`
      | status =~ "4.."
      [10m]
  )
)
```

## Grafana Dashboard Queries

```
# Variable: IP to filter
# In dashboard variable settings:
# Query: label_values({job="nginx"} | regexp `^(?P<client_ip>\S+)`, client_ip)

# Panel: Logs from selected IP
{job="nginx"} |= "${selected_ip}"

# Panel: Request rate for selected IP
rate({job="nginx"} |= "${selected_ip}" [1m])
```

## LogQL Cheat Sheet for IPv4

```logql
# Exact match
{job="nginx"} |= "192.168.1.100"

# Subnet match (line filter)
{job="nginx"} |~ "^10\\.1\\."

# After parsing — use label filter
{job="nginx"} | json | remote_addr = "203.0.113.42"

# Error rate from IP
rate({job="nginx"} |= "203.0.113.42" | status_code >= 400 [5m])
```

## Conclusion

Loki LogQL queries IPv4 addresses with line filters (`|=`, `|~`) for fast pattern matching, or structured parsers (JSON, regexp, pattern) to extract IPs as labels for aggregation. Use `rate()` with parsed IP labels to build per-IP request rate dashboards in Grafana, and create a Grafana variable with `label_values()` to make the IP filter interactive.
