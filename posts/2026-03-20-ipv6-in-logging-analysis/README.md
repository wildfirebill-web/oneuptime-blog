# How to Handle IPv6 in Logging and Log Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Logging, Log Analysis, Nginx, Apache, ELK Stack, Monitoring

Description: Handle IPv6 addresses correctly in application logs, web server access logs, and log analysis pipelines including normalization, parsing, and Elasticsearch indexing.

## Introduction

IPv6 addresses in logs present challenges for parsing, storage, and analysis. A single IPv6 host may appear in logs under multiple equivalent representations, making unique visitor counts inaccurate and IP-based queries difficult. This guide covers logging configuration and log analysis for IPv6.

## Nginx: Logging IPv6 Addresses

Nginx logs IPv6 addresses correctly by default. Verify your log format:

```nginx
# /etc/nginx/nginx.conf
http {
    # Default combined log format - works with IPv6
    log_format combined '$remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent"';

    # For dual-stack, $remote_addr shows either IPv4 or IPv6
    access_log /var/log/nginx/access.log combined;
}
```

IPv6 addresses in Nginx logs appear without brackets:
```
2001:db8::1 - - [19/Mar/2026:12:00:00 +0000] "GET / HTTP/1.1" 200 1234 "-" "curl/7.88.1"
```

## Apache: Logging IPv6 Addresses

Apache logs IPv6 addresses in brackets for the Combined Log Format:

```apache
# /etc/apache2/apache2.conf
LogFormat "%h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined

# %h logs the remote host (IP address)
# For IPv6, Apache may log: [2001:db8::1] or 2001:db8::1
# depending on configuration
```

Enable `mod_remoteip` for proper proxy handling:

```apache
LoadModule remoteip_module modules/mod_remoteip.so
RemoteIPHeader X-Forwarded-For
RemoteIPTrustedProxy 10.0.0.0/8 2001:db8::/32
```

## Normalizing IPv6 in Log Parsing Scripts

```python
import ipaddress
import re
from typing import Optional

def normalize_ip_in_log_line(line: str) -> str:
    """
    Normalize all IP addresses in a log line to compressed IPv6 form.
    Handles both IPv4 and IPv6 addresses.
    """
    # Pattern to match IPv6 in brackets or standalone
    ipv6_pattern = r'\[?([0-9a-fA-F:]+:[0-9a-fA-F:]*)\]?'

    def normalize_match(m):
        addr_str = m.group(1)
        try:
            addr = ipaddress.ip_address(addr_str)
            if isinstance(addr, ipaddress.IPv6Address):
                return str(addr)  # Compressed form
        except ValueError:
            pass
        return m.group(0)  # Return original if not valid

    return re.sub(ipv6_pattern, normalize_match, line)

def extract_ip_from_nginx_log(line: str) -> Optional[str]:
    """Extract and normalize IP from Nginx access log line."""
    parts = line.split(' ', 1)
    if not parts:
        return None

    raw_ip = parts[0].strip('[]')
    # Strip zone ID
    raw_ip = raw_ip.split('%')[0]

    try:
        # Normalize to standard form
        addr = ipaddress.ip_address(raw_ip)
        # Convert IPv4-mapped to pure IPv4
        if hasattr(addr, 'ipv4_mapped') and addr.ipv4_mapped:
            return str(addr.ipv4_mapped)
        return str(addr)
    except ValueError:
        return raw_ip

# Process a log file
with open('/var/log/nginx/access.log') as f:
    for line in f:
        ip = extract_ip_from_nginx_log(line)
        if ip:
            print(f"Client IP: {ip}")
```

## Elasticsearch/ELK: Indexing IPv6 Addresses

Configure Elasticsearch to properly handle IPv6 addresses:

```json
// Elasticsearch index mapping for IP logging
PUT /access-logs
{
  "mappings": {
    "properties": {
      "client_ip": {
        "type": "ip"  // Elasticsearch ip type handles IPv4 and IPv6
      },
      "timestamp": {
        "type": "date"
      },
      "method": {
        "type": "keyword"
      },
      "path": {
        "type": "keyword"
      },
      "status": {
        "type": "integer"
      }
    }
  }
}
```

Query for IPv6 traffic in Elasticsearch:

```json
// Find all requests from IPv6 clients
GET /access-logs/_search
{
  "query": {
    "term": {
      "client_ip": {
        "value": "2001:db8::1"
      }
    }
  }
}

// CIDR range query for IPv6 subnet
GET /access-logs/_search
{
  "query": {
    "term": {
      "client_ip": "2001:db8::/32"
    }
  }
}
```

## Logstash: Parsing IPv6 from Logs

```ruby
# Logstash pipeline configuration for IPv6-aware log parsing
filter {
  # Parse Nginx combined log format
  grok {
    match => {
      "message" => "%{IP:client_ip} - %{DATA:user} \[%{HTTPDATE:timestamp}\] \"%{WORD:method} %{DATA:path} HTTP/%{NUMBER:http_version}\" %{NUMBER:status} %{NUMBER:bytes}"
    }
  }

  # Normalize IPv6 addresses
  if [client_ip] {
    ruby {
      code => "
        require 'ipaddr'
        ip = event.get('client_ip').to_s.gsub(/[\[\]]/, '')
        begin
          normalized = IPAddr.new(ip).to_s
          event.set('client_ip_normalized', normalized)
          event.set('ip_version', IPAddr.new(ip).ipv6? ? 6 : 4)
        rescue
          event.set('client_ip_normalized', ip)
        end
      "
    }
  }
}
```

## grep/awk Analysis for IPv6 Logs

```bash
# Count unique IPv6 client addresses from Nginx logs
grep -oE '[0-9a-fA-F:]+:[0-9a-fA-F:]+' /var/log/nginx/access.log | \
    sort | uniq -c | sort -rn | head -20

# Count requests by IP version
awk '{
    ip = $1
    if (ip ~ /:/) { ipv6++ } else { ipv4++ }
}
END {
    print "IPv4:", ipv4
    print "IPv6:", ipv6
}' /var/log/nginx/access.log

# Find IPv6 addresses making the most requests
grep -oE '([0-9a-fA-F]{1,4}:){1,7}[0-9a-fA-F]{1,4}|::1' /var/log/nginx/access.log | \
    sort | uniq -c | sort -rn | head -10
```

## Conclusion

IPv6 log handling requires normalization to compressed form for consistent analysis, proper Elasticsearch ip type mapping for range queries, and grok patterns that accept both IPv4 and IPv6 addresses. Always normalize to compressed form during ingestion to ensure accurate deduplication and subnet-based queries in your log analysis pipeline.
