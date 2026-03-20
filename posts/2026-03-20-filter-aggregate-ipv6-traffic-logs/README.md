# How to Filter and Aggregate IPv6 Traffic Logs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Log Analysis, Aggregation, Python, awk

Description: Filter and aggregate IPv6 traffic log data using command-line tools, Python, and SQL to produce traffic reports by subnet, address type, and time period.

## Introduction

Log aggregation for IPv6 traffic involves grouping by address, subnet prefix, address category, and time window. Unlike IPv4, IPv6 addresses should be normalized before aggregation to avoid treating different representations of the same address as distinct entries. This guide covers command-line approaches for quick analysis and Python for programmatic aggregation.

## Command-Line Aggregation with awk and grep

```bash
#!/bin/bash
# aggregate_ipv6_logs.sh — Quick IPv6 log analysis

LOG_FILE="${1:-/var/log/nginx/access.log}"

echo "=== IPv6 Traffic Summary ==="
echo "File: $LOG_FILE"
echo ""

# Count total IPv6 requests
IPV6_COUNT=$(grep -cE '^\[?[0-9a-fA-F]{1,4}:[0-9a-fA-F:]+\]?' "$LOG_FILE" 2>/dev/null || echo 0)
echo "Total IPv6 requests: $IPV6_COUNT"

# Top 20 IPv6 source IPs
echo ""
echo "--- Top 20 IPv6 Source IPs ---"
grep -oE '^[0-9a-fA-F:]{3,39}' "$LOG_FILE" | \
    grep ":" | \
    sort | uniq -c | sort -rn | head -20

# IPv6 vs IPv4 split
echo ""
echo "--- IPv4 vs IPv6 Split ---"
awk '{
    ip = $1
    gsub(/[\[\]]/, "", ip)
    if (ip ~ /:/) { ipv6++ } else { ipv4++ }
}
END {
    total = ipv4 + ipv6
    printf "IPv4: %d (%.1f%%)\n", ipv4, (total>0 ? ipv4*100/total : 0)
    printf "IPv6: %d (%.1f%%)\n", ipv6, (total>0 ? ipv6*100/total : 0)
}' "$LOG_FILE"

# IPv6 requests per hour
echo ""
echo "--- IPv6 Requests Per Hour ---"
grep -E '^[0-9a-fA-F:]{3,39}' "$LOG_FILE" | \
    grep -oE '\[[0-9]{2}/[A-Za-z]+/[0-9]{4}:[0-9]{2}' | \
    grep -oE '[0-9]{2}$' | \
    sort | uniq -c | sort -k2 -n
```

## Python Aggregation Script

```python
#!/usr/bin/env python3
# aggregate_ipv6.py

import re
import ipaddress
from collections import defaultdict, Counter
from datetime import datetime

# Nginx access log regex
LOG_RE = re.compile(
    r'^(?P<ip>[\[\]0-9a-fA-F:.]+) - \S+ \[(?P<time>[^\]]+)\] '
    r'"(?P<method>\S+) (?P<path>\S+)[^"]*" (?P<status>\d+) (?P<bytes>\d+)'
)

def normalize_ip(raw: str) -> str | None:
    raw = raw.strip('[]').split('%')[0]
    try:
        return str(ipaddress.ip_address(raw))
    except ValueError:
        return None

def classify_ipv6(addr: str) -> str:
    try:
        ip = ipaddress.ip_address(addr)
        if ip.version != 6:
            return "ipv4"
        if ip.is_loopback:
            return "loopback"
        if ip.is_link_local:
            return "link_local"
        if ip.is_private:
            return "ula"
        return "global_unicast"
    except ValueError:
        return "invalid"

def get_prefix(addr: str, prefix_len: int = 48) -> str:
    """Return the /prefix_len network of an IPv6 address."""
    try:
        ip = ipaddress.ip_address(addr)
        if ip.version == 6:
            net = ipaddress.ip_network(f"{addr}/{prefix_len}", strict=False)
            return str(net)
        return addr
    except ValueError:
        return addr

def analyze_log(log_file: str):
    ip_counter = Counter()
    prefix_counter = Counter()
    type_counter = Counter()
    hourly_counter = defaultdict(Counter)
    error_ips = Counter()

    with open(log_file) as f:
        for line in f:
            m = LOG_RE.match(line)
            if not m:
                continue

            ip = normalize_ip(m.group("ip"))
            if not ip or ':' not in ip:
                continue  # Skip IPv4 for this analysis

            status = int(m.group("status"))
            hour = m.group("time")[:13]  # "20/Mar/2026:10"

            ip_counter[ip] += 1
            prefix_counter[get_prefix(ip)] += 1
            type_counter[classify_ipv6(ip)] += 1
            hourly_counter[hour][ip] += 1
            if status >= 400:
                error_ips[ip] += 1

    # Report
    print(f"\nTop 20 IPv6 Addresses:")
    for ip, count in ip_counter.most_common(20):
        print(f"  {count:6d}  {ip}")

    print(f"\nTop 10 IPv6 /48 Prefixes:")
    for prefix, count in prefix_counter.most_common(10):
        print(f"  {count:6d}  {prefix}")

    print(f"\nAddress Type Distribution:")
    for t, count in type_counter.most_common():
        print(f"  {count:6d}  {t}")

    print(f"\nTop Error Sources (4xx/5xx):")
    for ip, count in error_ips.most_common(10):
        print(f"  {count:6d}  {ip}")

if __name__ == "__main__":
    import sys
    log_file = sys.argv[1] if len(sys.argv) > 1 else "/var/log/nginx/access.log"
    analyze_log(log_file)
```

## SQL Aggregation (for Loki/ClickHouse/DuckDB)

```sql
-- DuckDB / ClickHouse: aggregate IPv6 traffic logs

-- Top IPv6 prefixes by request count (simulate /32 prefix by truncating address)
SELECT
    regexp_replace(client_ip, ':[^:]+:[^:]+:[^:]+:[^:]+$', '::') AS prefix_48,
    COUNT(*) AS request_count,
    COUNT(DISTINCT client_ip) AS unique_ips,
    SUM(bytes) AS total_bytes
FROM nginx_logs
WHERE client_ip LIKE '%:%'
GROUP BY 1
ORDER BY request_count DESC
LIMIT 20;

-- Hourly IPv6 traffic trend
SELECT
    DATE_TRUNC('hour', timestamp) AS hour,
    COUNT(*) FILTER (WHERE client_ip LIKE '%:%') AS ipv6_requests,
    COUNT(*) FILTER (WHERE client_ip NOT LIKE '%:%') AS ipv4_requests
FROM nginx_logs
GROUP BY 1
ORDER BY 1;
```

## Conclusion

IPv6 log aggregation requires normalization before grouping to ensure address format variants map to the same key. Python's `ipaddress` module handles normalization reliably and also enables subnet grouping via `ip_network()` with `strict=False`. For command-line analysis of existing logs, `grep` for colon-containing addresses to isolate IPv6 traffic, then pipe through `sort | uniq -c | sort -rn` for quick frequency analysis. Use `/48` prefix grouping to identify organizational-level traffic patterns since IPv6 allocations typically assign a /48 per site.
