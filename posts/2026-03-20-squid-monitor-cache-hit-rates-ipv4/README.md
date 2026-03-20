# How to Monitor Squid Cache Hit Rates for IPv4 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Squid, Monitoring, Cache Hit Rate, IPv4, Squidclient, Metrics, Performance

Description: Learn how to monitor Squid proxy cache hit rates and performance metrics for IPv4 traffic using the cache manager interface and access log analysis.

---

Cache hit rate is the primary metric for measuring Squid's effectiveness. A high hit rate (>40%) means Squid is successfully serving content from its local cache, reducing upstream bandwidth consumption. This guide shows how to monitor hit rates in real time and through log analysis.

## Method 1: Cache Manager (squidclient)

The `squidclient` tool queries Squid's built-in cache manager interface.

```bash
# Enable cache manager access in squid.conf (allow from localhost by default)

# No extra configuration needed for local access

# Get overall cache statistics
squidclient -h localhost -p 3128 mgr:info

# Get detailed per-request counters
squidclient -h localhost -p 3128 mgr:counters
```

Key metrics from `mgr:info`:

```text
Cache information for squid:
    Hits as % of all requests:   5min: 42.3%, 60min: 38.7%
    Hits as % of bytes sent:     5min: 61.2%, 60min: 57.8%
    Memory hits as % of all requests: 5min: 18.1%
    Disk hits as % of all requests:   5min: 24.2%
```

## Method 2: Parsing the Access Log

```bash
# Count TCP_HIT vs TCP_MISS from the access log (last 1000 lines)
tail -1000 /var/log/squid/access.log | \
  awk '{print $4}' | \
  sort | uniq -c | sort -rn

# Calculate cache hit percentage from the full log
awk '{
  total++
  if ($4 ~ /HIT/) hits++
} END {
  printf "Total: %d, Hits: %d, Hit Rate: %.1f%%\n", total, hits, (hits/total)*100
}' /var/log/squid/access.log
```

## Squid Access Log Result Codes

| Code | Meaning |
|------|---------|
| `TCP_HIT` | Served from disk cache |
| `TCP_MEM_HIT` | Served from memory cache |
| `TCP_MISS` | Cache miss; fetched from origin |
| `TCP_REFRESH_HIT` | Revalidated with origin; served from cache |
| `TCP_DENIED` | Access denied by ACL |
| `TCP_TUNNEL` | HTTPS CONNECT tunnel |

## Method 3: cachemgr.cgi Web Interface

Squid includes a CGI-based cache manager for browser-based monitoring.

```bash
# Install the cache manager CGI
apt install squid-cgi -y

# Configure Apache or Nginx to serve it
# (usually at /usr/lib/cgi-bin/cachemgr.cgi)

# Or access directly via squidclient in a loop
watch -n5 "squidclient -h localhost -p 3128 mgr:info | grep 'Hits as %'"
```

## Enabling Detailed Logging

```squid
# /etc/squid/squid.conf
# Add the result code to the access log format for better analysis
logformat combined %ts.%03tu %6tr %>a %Ss/%03>Hs %<st %rm %ru %[un %Sh/%<a %mt
access_log /var/log/squid/access.log combined
```

## Real-Time Hit Rate Dashboard (Shell Script)

```bash
#!/bin/bash
# monitor_squid.sh - Print cache hit rate every 10 seconds
while true; do
    STATS=$(squidclient -h localhost -p 3128 mgr:info 2>/dev/null)
    HIT5=$(echo "$STATS" | grep "5min:" | head -1 | grep -oP '\d+\.\d+(?=%)')
    echo "$(date '+%H:%M:%S') - 5-min cache hit rate: ${HIT5}%"
    sleep 10
done
```

## Key Takeaways

- `squidclient mgr:info` gives real-time hit rates for 5-minute and 60-minute windows.
- Parse the access log and count `TCP_HIT` vs `TCP_MISS` for historical analysis.
- A hit rate below 20% suggests the cache is too small, TTLs are too short, or traffic is mostly dynamic/uncacheable.
- Increase `cache_mem` and `cache_dir` size, or tune `minimum_object_size` / `maximum_object_size`, to improve hit rates.
