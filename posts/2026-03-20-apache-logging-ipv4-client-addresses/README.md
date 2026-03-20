# How to Configure Apache Logging to Record IPv4 Client Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Apache, Logging, IPv4, Access Logs, CustomLog, Security

Description: Configure Apache custom log formats to accurately record client IPv4 addresses, including real IPs from X-Forwarded-For headers when behind a proxy.

## Introduction

Apache's access logs are your primary source for security auditing, traffic analysis, and debugging. Properly capturing client IPv4 addresses-especially when Apache sits behind load balancers-requires understanding log format directives and the `mod_remoteip` module.

## Default Log Formats

Apache ships with two built-in formats:

```apache
# combined: includes referer and user agent

LogFormat "%h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined

# common: basic format
LogFormat "%h %l %u %t \"%r\" %>s %O" common

# %h = hostname or IP of the remote client
# %l = ident (usually -)
# %u = authenticated user (usually -)
# %t = time of request
# %r = first line of request
# %>s = final status code
# %O = bytes sent
```

## Enabling IP Logging (Not Hostname Resolution)

Disable hostname lookups to log raw IPv4 addresses (faster):

```apache
# /etc/apache2/apache2.conf or httpd.conf

# Disable reverse DNS lookups - log IP addresses directly
HostnameLookups Off
```

## Custom Log Format with IPv4 Focus

```apache
# /etc/apache2/conf-available/logging.conf

# Detailed format: client IP, forwarded IPs, timing, upstream
LogFormat "%a %{c}a %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\" %D" detailed
# %a  = client IP (real IP after mod_remoteip processing)
# %{c}a = underlying connection IP (load balancer IP)
# %D  = time to serve in microseconds

# JSON format for log aggregation systems
LogFormat "{ \"time\": \"%{%Y-%m-%dT%H:%M:%S}t\", \"client_ip\": \"%a\", \"connection_ip\": \"%{c}a\", \"method\": \"%m\", \"uri\": \"%U\", \"status\": %>s, \"bytes\": %O, \"duration_us\": %D }" json_access
```

## Logging Real Client IP Behind a Load Balancer

Enable `mod_remoteip` to unwrap `X-Forwarded-For`:

```bash
sudo a2enmod remoteip
```

```apache
# /etc/apache2/conf-available/remoteip.conf
RemoteIPHeader X-Forwarded-For
RemoteIPTrustedProxy 10.0.0.1    # Your load balancer
RemoteIPTrustedProxy 10.0.0.2
```

Now `%a` in log format shows the real client IPv4, while `%{c}a` shows the load balancer's IP.

## Per-VirtualHost Log Configuration

```apache
<VirtualHost *:80>
    ServerName api.example.com

    # Use detailed format for API access log
    CustomLog /var/log/apache2/api-access.log detailed

    # Log 4xx and 5xx errors separately for monitoring
    CustomLog /var/log/apache2/api-errors.log combined "expr=%{REQUEST_STATUS} >= 400"
</VirtualHost>
```

## Conditional Logging

Skip logging health checks and internal pings:

```apache
# Define environment variable for requests to skip
SetEnvIf Request_URI "^/health$" no_log
SetEnvIf Request_URI "^/ping$"   no_log

# Log all requests EXCEPT those with no_log set
CustomLog /var/log/apache2/access.log combined env=!no_log
```

## Analyzing Client IPv4 from Logs

```bash
# Top 10 client IPs by request count
awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn | head 10

# All requests from a specific IP
grep '^203\.0\.113\.50' /var/log/apache2/access.log

# Count error responses per client IP
awk '$9 ~ /^[45]/{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn | head 20

# Requests per minute (for rate analysis)
awk '{print $4}' /var/log/apache2/access.log | cut -c2-18 | uniq -c
```

## Log Rotation

```bash
# Apache uses logrotate - check configuration
cat /etc/logrotate.d/apache2

# Typical logrotate config:
# /var/log/apache2/*.log {
#     daily
#     missingok
#     rotate 14
#     compress
#     delaycompress
#     notifempty
#     create 640 root adm
#     sharedscripts
#     postrotate
#         invoke-rc.d apache2 reload > /dev/null 2>&1 || true
#     endscript
# }
```

## Conclusion

Accurate IPv4 logging in Apache requires `HostnameLookups Off` for speed, `mod_remoteip` for real-IP extraction behind proxies, and custom `LogFormat` directives using `%a` for the processed client IP. Use JSON formats for modern log aggregation and conditional logging to reduce noise from health check endpoints.
