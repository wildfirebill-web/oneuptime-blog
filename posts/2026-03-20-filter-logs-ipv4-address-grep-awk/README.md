# How to Filter Logs by IPv4 Address Using grep and awk

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, IPv4, Log Analysis, grep, awk, Shell, Security

Description: Filter log files by IPv4 address using grep and awk one-liners, including exact match, CIDR range filtering, subnet filtering, and multi-file log analysis.

## Introduction

Filtering logs by IPv4 address is a daily task for operators. `grep` handles exact and pattern matches; `awk` provides field-based extraction and arithmetic for range comparisons. Combining them covers most log analysis scenarios.

## Exact IP Match with grep

```bash
# Find all log lines for a specific IP
grep "^203.0.113.42 " /var/log/nginx/access.log

# Anywhere in the line (e.g., syslog format)
grep "203\.0\.113\.42" /var/log/syslog

# Multiple IPs
grep -E "203\.0\.113\.42|198\.51\.100\.5" /var/log/nginx/access.log

# Case-insensitive search (for hex representations)
grep -i "c0a80101" /var/log/app.log
```

## IPv4 Pattern Matching

```bash
# Match any IPv4 address in a log line
IPV4_PATTERN='[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}'
grep -o "$IPV4_PATTERN" /var/log/nginx/access.log | sort -u

# Extended regex version
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

## Filter by /24 Subnet with grep

```bash
# All requests from 192.168.1.x
grep "^192\.168\.1\." /var/log/nginx/access.log

# Any subnet: replace last octet with wildcard
grep -E "^10\.1\.[0-9]+\.[0-9]+" /var/log/syslog
```

## awk for CIDR Range Filtering

```bash
# Filter lines where first field is in 10.1.0.0/16 (10.1.x.x)
awk -F'[. ]' '
  $1 == 10 && $2 == 1 {print}
' /var/log/nginx/access.log

# More precise — check all four octets
awk '
{
  split($1, ip, ".");
  if (ip[1]+0==10 && ip[2]+0==1 && ip[3]+0>=0 && ip[3]+0<=255)
    print
}
' /var/log/nginx/access.log
```

## Python CIDR Filter (for /22, /20, etc.)

```bash
python3 - << 'PYEOF'
import ipaddress, sys

target_net = ipaddress.IPv4Network("10.64.0.0/20")

with open("/var/log/nginx/access.log") as f:
    for line in f:
        ip_str = line.split()[0]
        try:
            if ipaddress.IPv4Address(ip_str) in target_net:
                print(line, end="")
        except ValueError:
            pass
PYEOF
```

## Multi-File and Compressed Log Search

```bash
# Search across rotated logs
grep "203\.0\.113\.42" /var/log/nginx/access.log*

# Search gzipped logs
zgrep "203\.0\.113\.42" /var/log/nginx/access.log.*.gz

# Search all access logs recursively
find /var/log -name "access.log*" -exec grep "203\.0\.113\.42" {} /dev/null \;
```

## Count Requests per Minute from IP

```bash
# Count requests from 203.0.113.42, grouped by minute
awk '/^203\.0\.113\.42/ {
  match($4, /\[([0-9:\/]+):[0-9]+:[0-9]+/, ts)
  print ts[1]
}' /var/log/nginx/access.log | sort | uniq -c
```

## Conclusion

`grep` with a fixed-string IP or regex pattern is fastest for exact and subnet searches. Use `awk` field splitting for CIDR comparisons involving multiple octets. For precise /20 or /22 subnet filtering, a short Python snippet using `ipaddress.IPv4Network` handles the bit math correctly. Always search rotated and compressed logs with `*` glob and `zgrep`.
