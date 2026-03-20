# How to Parse IPv4 Addresses from Nginx Access Logs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, IPv4, Log Analysis, Linux, Shell, Python, Security

Description: Extract and analyze IPv4 addresses from Nginx access logs using shell commands, awk, Python, and GoAccess to identify top clients, block attackers, and monitor traffic patterns.

## Introduction

Nginx access logs record the client's IPv4 (or IPv6) address as the first field in the default combined log format. Parsing these addresses enables traffic analysis, abuse detection, and security incident response.

## Nginx Log Format

```
# Default combined format:
# $remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"

Example:
203.0.113.42 - - [20/Mar/2026:10:15:30 +0000] "GET /api/users HTTP/1.1" 200 1234 "-" "Mozilla/5.0"
```

## Extract All Client IPs with awk

```bash
# Print only IP addresses
awk '{print $1}' /var/log/nginx/access.log

# Top 10 IPs by request count
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Sample output:
#   4523 203.0.113.42
#   2341 198.51.100.5
#    892 10.1.1.100
```

## Count Requests per IP in a Time Window

```bash
# Extract IPs from a specific date
grep "20/Mar/2026" /var/log/nginx/access.log | awk '{print $1}' | \
  sort | uniq -c | sort -rn | head -20
```

## Identify 4xx/5xx Errors by IP

```bash
# IPs generating 404 errors
awk '$9 == 404 {print $1}' /var/log/nginx/access.log | \
  sort | uniq -c | sort -rn | head -10

# IPs with 5xx errors
awk '$9 ~ /^5/ {print $1}' /var/log/nginx/access.log | \
  sort | uniq -c | sort -rn
```

## Python Log Parser

```python
import re
from collections import Counter
from pathlib import Path

LOG_FILE = "/var/log/nginx/access.log"
IPV4_RE  = re.compile(r'^(\d{1,3}\.){3}\d{1,3}')

def parse_nginx_log(log_file: str) -> dict:
    ip_counter = Counter()
    status_by_ip: dict[str, Counter] = {}

    with open(log_file) as f:
        for line in f:
            parts = line.split()
            if len(parts) < 9:
                continue
            ip     = parts[0]
            status = parts[8]

            if not IPV4_RE.match(ip):
                continue

            ip_counter[ip] += 1
            if ip not in status_by_ip:
                status_by_ip[ip] = Counter()
            status_by_ip[ip][status] += 1

    return {"counts": ip_counter.most_common(20), "by_status": status_by_ip}

results = parse_nginx_log(LOG_FILE)
print("Top 20 IPs:")
for ip, count in results["counts"]:
    errors = sum(v for k, v in results["by_status"][ip].items() if k.startswith(('4','5')))
    print(f"  {ip:<20} requests={count:5d}  errors={errors}")
```

## Block High-Frequency IPs with fail2ban

```ini
# /etc/fail2ban/jail.d/nginx-req-limit.conf
[nginx-req-limit]
enabled  = true
filter   = nginx-req-limit
action   = iptables-multiport[name=ReqLimit, port="http,https"]
logpath  = /var/log/nginx/access.log
maxretry = 100
findtime = 60
bantime  = 3600
```

## Conclusion

Nginx access log IPv4 parsing is straightforward with `awk '{print $1}'`. For security analysis, filter by HTTP status codes to find abusive clients. Python provides richer analysis with per-IP status breakdowns. Feed top offending IPs into fail2ban or firewall rules for automated blocking.
