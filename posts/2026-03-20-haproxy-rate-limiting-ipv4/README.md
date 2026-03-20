# How to Configure HAProxy Rate Limiting by IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: HAProxy, Rate Limiting, IPv4, Security, Stick Tables, DDoS Protection

Description: Implement per-IPv4 rate limiting in HAProxy using stick tables to protect backend servers from traffic floods, brute force attacks, and API abuse.

## Introduction

HAProxy implements rate limiting via stick tables that track per-client metrics. Unlike external rate limiters, this approach runs in HAProxy's event loop at line speed with no external dependencies.

## Basic Request Rate Limiting

```haproxy
# /etc/haproxy/haproxy.cfg

frontend http_in
    bind 203.0.113.10:80
    mode http

    # Stick table: track HTTP request rate per source IP over 10-second window
    stick-table type ip size 100k expire 60s store http_req_rate(10s)

    # Track each incoming request against the source IP
    http-request track-sc0 src

    # Reject if more than 100 requests per 10 seconds from single IP
    http-request deny deny_status 429 \
        if { sc_http_req_rate(0) gt 100 }

    default_backend app_servers
```

## Multi-Tier Rate Limiting

Apply different limits for different endpoint types:

```haproxy
frontend api_in
    bind 203.0.113.10:443 ssl crt /etc/haproxy/certs/bundle.pem

    # Tables for different metrics
    stick-table type ip size 100k expire 60s store http_req_rate(60s),http_err_rate(30s),conn_rate(10s)
    http-request track-sc0 src

    # Connection rate: max 20 new connections per 10 seconds
    acl too_many_connections    sc_conn_rate(0) gt 20
    # Request rate: max 300 per minute
    acl too_many_requests       sc_http_req_rate(0) gt 300
    # Error rate: max 30 errors per 30 seconds (likely attack/scan)
    acl too_many_errors         sc_http_err_rate(0) gt 30

    # Block API abuse
    http-request deny deny_status 429 if too_many_requests
    http-request deny deny_status 503 if too_many_connections
    http-request deny deny_status 429 if too_many_errors

    # Stricter limits for login endpoint
    acl is_login path /api/auth/login

    # Track login requests separately
    stick-table type ip size 50k expire 5m store http_req_rate(60s) id login_table
    http-request track-sc1 src if is_login

    # Max 10 login attempts per minute per IP
    http-request deny deny_status 429 if is_login { sc_http_req_rate(1) gt 10 }

    default_backend api_servers
```

## Whitelisting IPs from Rate Limits

```haproxy
frontend http_in
    bind 203.0.113.10:80

    stick-table type ip size 100k expire 60s store http_req_rate(10s)

    acl is_trusted_network src 10.0.0.0/8 192.168.0.0/16

    # Only track non-trusted IPs
    http-request track-sc0 src if !is_trusted_network

    # Apply rate limit only to tracked (untrusted) IPs
    http-request deny deny_status 429 if !is_trusted_network { sc_http_req_rate(0) gt 100 }

    default_backend app_servers
```

## Return Custom Rate Limit Headers

Send RFC 6585 standard headers when rate limiting:

```haproxy
frontend http_in
    bind 203.0.113.10:80

    stick-table type ip size 100k expire 60s store http_req_rate(60s)
    http-request track-sc0 src

    # Add remaining rate info as headers on all responses
    http-response set-header X-RateLimit-Limit 300
    http-response set-header Retry-After 60 if { sc_http_req_rate(0) gt 300 }

    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 300 }

    default_backend app_servers
```

## Monitoring Rate Limit Activity

```bash
# View stick table to see per-IP counters

echo "show table http_in" | sudo socat stdio /run/haproxy/admin.sock

# Sample output:
# # table: http_in, type: ip, size:102400, used:3
# 0x55c1234: key=203.0.113.50 use=0 exp=45000 http_req_rate(10000)=87

# Find IPs exceeding rate limits
echo "show table http_in" | sudo socat stdio /run/haproxy/admin.sock | \
  awk '$NF > 100 {print $4, $NF}'
```

## Conclusion

HAProxy rate limiting via stick tables is efficient, accurate, and requires no external dependencies. Combine `http_req_rate`, `conn_rate`, and `http_err_rate` counters to catch different attack patterns. Whitelist internal networks, apply stricter limits to sensitive endpoints like login forms, and monitor stick table contents in real-time to tune limits for your traffic patterns.
