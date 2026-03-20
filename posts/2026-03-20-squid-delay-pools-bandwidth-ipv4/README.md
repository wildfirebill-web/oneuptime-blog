# How to Set Up Squid Delay Pools for Bandwidth Limiting by IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Squid, Delay Pools, IPv4, Bandwidth, QoS, Rate Limiting

Description: Configure Squid delay pools to limit bandwidth consumption per IPv4 address or subnet, implementing QoS policies that prevent individual users from saturating shared links.

## Introduction

Squid delay pools throttle download speeds for specific clients or content types. Rather than dropping connections, delay pools add artificial latency to enforce bandwidth limits—useful for corporate proxies where fair-use policies need enforcement.

## Delay Pool Types

| Class | Description |
|---|---|
| Class 1 | One aggregate pool for all traffic |
| Class 2 | Aggregate pool + per-subnet pool |
| Class 3 | Aggregate + per-subnet + per-host pool |

## Class 3 Delay Pool (Per-Host Limiting)

```bash
# /etc/squid/squid.conf

http_port 10.0.0.1:3128

# Access control
acl internal src 10.0.0.0/8
http_access allow internal
http_access deny all

# Define how many delay pools
delay_pools 2

# Pool 1: Normal users - bandwidth limited
delay_class 1 3    # Class 3: aggregate + subnet + host limits

# Pool 1 parameters:
# delay_parameters <pool> <aggregate> <network> <individual>
# Format: rate/max_burst (bytes/s / max bytes)
# -1/-1 = unlimited
delay_parameters 1 \
    10240000/10240000 \   # Aggregate: 10 MB/s total for all traffic
    5120000/5120000 \     # Per /24 subnet: 5 MB/s
    102400/204800         # Per host: 100 KB/s, burst to 200 KB

# Apply pool 1 to internal clients (except fast_users)
acl slow_users src 10.0.0.0/8

# Pool 2: Unlimited for specific hosts
delay_class 2 3
delay_parameters 2 -1/-1 -1/-1 -1/-1   # All unlimited

acl fast_users src 10.0.1.100 10.0.1.101  # Admins/servers

delay_access 2 allow fast_users    # No delay for fast_users
delay_access 2 deny all

delay_access 1 allow slow_users    # Apply limit to everyone else
delay_access 1 deny all
```

## Limiting by Content Type

Throttle large downloads but not web browsing:

```bash
delay_pools 1
delay_class 1 2    # Class 2: aggregate + subnet

# Limit to 1 MB/s aggregate for large files
delay_parameters 1 1048576/2097152 524288/1048576

# ACL for large/slow content types
acl large_downloads url_regex -i \.(iso|mp4|mkv|zip|tar|gz|rar)$

delay_access 1 allow large_downloads
delay_access 1 deny all
```

## Testing Bandwidth Limits

```bash
# Download a large file through proxy and measure speed
curl -x http://10.0.0.1:3128 -o /dev/null http://speedtest.example.com/100MB.bin \
  --progress-bar 2>&1

# Expected: transfer rate should be throttled to ~100 KB/s per host

# Check Squid stats
squidclient -h 127.0.0.1 mgr:delay | grep pool

# Verify delay pool is active
sudo tail -f /var/log/squid/cache.log | grep delay
```

## Conclusion

Squid delay pools implement bandwidth throttling without dropping connections. Class 3 pools provide three tiers of limits: aggregate (total), per-subnet, and per-host. Set rates in bytes-per-second with burst allowances, apply different pools to different client groups via `delay_access`, and use `-1/-1` for unlimited tiers. Test with large file downloads to verify throttling behavior matches your policy.
