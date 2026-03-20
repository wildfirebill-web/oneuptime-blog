# How to Log IPv6 Client Addresses in Apache

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Apache, Logging, Access Logs, CustomLog

Description: Learn how to log IPv6 client addresses in Apache access logs, customize log formats for IPv6 analysis, and handle IPv4-mapped addresses from dual-stack configurations.

## Default Apache Logging for IPv6

Apache automatically logs IPv6 client addresses with the default combined log format:

```apache
# Default combined log format

LogFormat "%h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined

# %h = client IP address (IPv4 or IPv6)
# With IPv6 dual-stack, %h shows the real IPv6 address
```

## Sample IPv6 Access Log Entries

```text
# IPv6 access log entries look like:
2001:db8::10 - - [20/Mar/2026:10:00:00 +0000] "GET / HTTP/1.1" 200 1234 "-" "Mozilla/5.0"
::1 - - [20/Mar/2026:10:00:01 +0000] "GET /health HTTP/1.1" 200 12 "-" "curl/7.88.1"

# IPv4-mapped (when ipv6only is off):
::ffff:192.168.1.10 - - [20/Mar/2026:10:00:02 +0000] "GET / HTTP/1.1" 200 1234 "-" "..."
```

## Custom Log Format with IP Version

```apache
# Add IP version indicator to logs
<IfModule log_config_module>
    # IPv6-enhanced log format
    LogFormat "%h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\" %{IPV}n" combined_ipv6

    # Set environment variable based on remote address
    SetEnvIf Remote_Addr ":" IPV=6
    SetEnvIf Remote_Addr "^[0-9]" IPV=4

    CustomLog ${APACHE_LOG_DIR}/access.log combined_ipv6
</IfModule>
```

## Separate Log Files for IPv4 and IPv6

```apache
<VirtualHost *:80>
    ServerName example.com

    # Set environment for IPv6 detection
    SetEnvIf Remote_Addr ":" IS_IPV6
    SetEnvIf Remote_Addr "^[0-9]" IS_IPV4

    # Separate access logs
    CustomLog ${APACHE_LOG_DIR}/access-ipv4.log combined env=IS_IPV4
    CustomLog ${APACHE_LOG_DIR}/access-ipv6.log combined env=IS_IPV6

    DocumentRoot /var/www/example
</VirtualHost>
```

## Log with X-Forwarded-For (When Behind Proxy)

```apache
# When behind a load balancer, enable mod_remoteip for real client IP
<IfModule mod_remoteip.c>
    RemoteIPHeader X-Forwarded-For

    # Trust IPv6 load balancer addresses
    RemoteIPTrustedProxy 2001:db8:lb::/64
    RemoteIPTrustedProxy 192.168.1.0/24
</IfModule>

# After mod_remoteip, %h logs the real client IP
LogFormat "%a %l %u %t \"%r\" %>s %O" combined_real
# %a = Real IP (after mod_remoteip processing)
# %{c}a = Connection IP (actual TCP peer, before mod_remoteip)
```

## Analyze IPv6 Logs

```bash
# Count requests by IPv6 prefix (/64)
awk '{print $1}' /var/log/apache2/access.log | \
    grep ':' | \
    python3 -c "
import sys
import ipaddress
from collections import Counter
c = Counter()
for line in sys.stdin:
    addr = line.strip().strip('::ffff:')
    try:
        if ':' in addr:
            net = ipaddress.IPv6Network(addr + '/64', strict=False)
            c[str(net)] += 1
    except: pass
for net, count in c.most_common(20):
    print(count, net)
"

# Find most active IPv6 addresses
awk '{print $1}' /var/log/apache2/access.log | \
    grep ':' | sort | uniq -c | sort -rn | head -20

# Count IPv4 vs IPv6 requests
echo "IPv4: $(awk '{print $1}' /var/log/apache2/access.log | grep -c '^[0-9]')"
echo "IPv6: $(awk '{print $1}' /var/log/apache2/access.log | grep -c ':')"
```

## Summary

Apache logs IPv6 client addresses automatically via `%h` in log formats. IPv4-mapped addresses (`::ffff:192.168.1.1`) appear when using `ipv6only=off`. Use `SetEnvIf Remote_Addr ":"` to detect IPv6 clients and route to separate log files or add IP version to log format. When behind a proxy, use `mod_remoteip` with `RemoteIPTrustedProxy 2001:db8:lb::/64` to log real client IPs. Use `%a` (after remoteip processing) vs `%{c}a` (connection IP) to distinguish.
