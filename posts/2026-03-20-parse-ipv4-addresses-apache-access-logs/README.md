# How to Parse IPv4 Addresses from Apache Access Logs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Apache, IPv4, Log Analysis, Linux, Shell, Python, Security

Description: Extract and analyze IPv4 addresses from Apache HTTP Server access logs using shell tools, awk, Python, and regular expressions to identify traffic patterns and security events.

## Introduction

Apache access logs use the Combined Log Format by default, with the client IPv4 address as the first field. Parsing these logs reveals top clients, error sources, and suspicious activity patterns.

## Apache Log Format

```text
# Combined Log Format (default):

# %h %l %u %t "%r" %>s %O "%{Referer}i" "%{User-Agent}i"

Example:
203.0.113.42 - frank [20/Mar/2026:10:15:30 -0700] "GET /index.html HTTP/1.1" 200 2326 "-" "Mozilla/5.0"
```

## Basic Shell Analysis

```bash
# Top 10 IPs by request count
awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn | head -10

# Count unique IPs today
awk '{print $1}' /var/log/apache2/access.log | sort -u | wc -l

# IPs with 4xx errors
awk '$9 ~ /^4/ {print $1}' /var/log/apache2/access.log | \
  sort | uniq -c | sort -rn | head -20
```

## Filter by Method and Status

```bash
# POST requests and their source IPs
awk '$6 == "\"POST" {print $1}' /var/log/apache2/access.log | \
  sort | uniq -c | sort -rn | head -10

# Bandwidth consumers by IP
awk '{bytes[$1]+=$10} END {for(ip in bytes) print bytes[ip], ip}' \
  /var/log/apache2/access.log | sort -rn | head -10
```

## Python Parser with Regex

```python
import re
from collections import defaultdict, Counter

LOG_FILE = "/var/log/apache2/access.log"

# Apache Combined Log Format
LOG_RE = re.compile(
    r'^(?P<ip>\S+) \S+ \S+ \[(?P<time>[^\]]+)\] '
    r'"(?P<method>\S+) (?P<path>\S+)[^"]*" '
    r'(?P<status>\d{3}) (?P<bytes>\S+)'
)

def analyze_apache_log(path: str):
    ip_requests = Counter()
    ip_bytes    = defaultdict(int)
    ip_errors   = Counter()

    with open(path) as f:
        for line in f:
            m = LOG_RE.match(line)
            if not m:
                continue
            ip     = m.group("ip")
            status = int(m.group("status"))
            size   = int(m.group("bytes")) if m.group("bytes") != "-" else 0

            ip_requests[ip] += 1
            ip_bytes[ip]    += size
            if status >= 400:
                ip_errors[ip] += 1

    print("Top IPs by requests:")
    for ip, count in ip_requests.most_common(10):
        mb = ip_bytes[ip] / 1_048_576
        print(f"  {ip:<20} req={count:5d}  data={mb:6.1f}MB  errors={ip_errors[ip]}")

analyze_apache_log(LOG_FILE)
```

## Detect Brute Force (Many 401s from Same IP)

```bash
# IPs with > 50 authentication failures
awk '$9 == 401 {print $1}' /var/log/apache2/access.log | \
  sort | uniq -c | sort -rn | awk '$1 > 50 {print $2}'
```

## Block Abusive IPs with mod_authz_host

```apache
# .htaccess or VirtualHost
<RequireAll>
    Require all granted
    Require not ip 203.0.113.0/24
    Require not ip 198.51.100.5
</RequireAll>
```

## Conclusion

Apache access log parsing with `awk '{print $1}'` extracts client IPv4 addresses instantly. For deeper analysis, use Python to build per-IP request counts, bandwidth usage, and error rates. Automate blocking of abusive IPs by combining log analysis with fail2ban or Apache's `mod_authz_host` directives.
