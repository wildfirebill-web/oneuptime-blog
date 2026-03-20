# How to Configure Apache Timeout and ProxyTimeout for IPv4 Backends

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Apache, Timeout, ProxyTimeout, IPv4, Reverse Proxy, Performance, mod_proxy

Description: Configure Apache's Timeout and ProxyTimeout directives to control how long requests wait for IPv4 backend responses, preventing hanging connections from exhausting server resources.

## Introduction

Apache's timeout settings control how long it waits for client requests, backend responses, and data transfers. Misconfigured timeouts can cause two problems: timeouts too low cause premature failures for legitimate slow requests; timeouts too high allow hanging connections to exhaust Apache's MaxRequestWorkers, making the server unresponsive.

## Apache Timeout Directives

| Directive | Controls | Default |
|-----------|---------|---------|
| `Timeout` | Max time for client to complete request | 60s |
| `ProxyTimeout` | Max time to wait for backend response | Uses `Timeout` |
| `ProxyConnectTimeout` | Timeout connecting to backend | 10s |
| `KeepAliveTimeout` | Idle keepalive connection timeout | 5s |

## Setting Global Timeout

```apache
# /etc/apache2/apache2.conf

# Time to receive the complete HTTP request from the client
# Covers: time to receive request headers + body
Timeout 30

# Keep-alive timeout for persistent connections
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5
```

## Configuring ProxyTimeout

`ProxyTimeout` sets the maximum time Apache waits for the backend (IPv4 server) to send the complete response:

```apache
# /etc/apache2/sites-available/myapp.conf

<VirtualHost *:80>
    ServerName app.example.com
    
    # Enable proxy modules
    ProxyPreserveHost On
    ProxyRequests Off
    
    # Timeout connecting to the backend server
    ProxyConnectTimeout 10
    
    # Timeout waiting for backend to start sending response
    # Set this higher than your longest legitimate request
    ProxyTimeout 120
    
    # Proxy all requests to the backend
    ProxyPass / http://10.0.0.5:8080/
    ProxyPassReverse / http://10.0.0.5:8080/
</VirtualHost>
```

## Per-Location Timeout Overrides

Set different timeouts for different endpoints:

```apache
<VirtualHost *:443>
    ServerName api.example.com
    
    # Default: short timeout for most API endpoints
    ProxyTimeout 30
    
    # Long-running report endpoint: allow up to 10 minutes
    <Location /api/reports>
        ProxyTimeout 600
    </Location>
    
    # File upload: allow large uploads
    <Location /api/upload>
        # Increase client request timeout for large bodies
        RequestReadTimeout body=600,minrate=1
        ProxyTimeout 600
    </Location>
    
    # Health check: short timeout only
    <Location /health>
        ProxyTimeout 5
    </Location>
    
    ProxyPass / http://10.0.0.5:8080/
    ProxyPassReverse / http://10.0.0.5:8080/
</VirtualHost>
```

## RequestReadTimeout for Slow Clients

`RequestReadTimeout` sets timeouts for the client-to-Apache phase:

```apache
# /etc/apache2/conf-available/request-timeout.conf

<IfModule mod_reqtimeout.c>
    # Client must send headers within 20s, body (if any) within 40s
    # minrate=500: give 500 bytes/s leeway for slow connections
    RequestReadTimeout header=20-40,minrate=500 body=20,minrate=500
</IfModule>
```

## Configuring Timeouts for mod_proxy_balancer

When using a balancer with multiple backends:

```apache
<Proxy "balancer://mycluster">
    BalancerMember http://10.0.0.5:8080 timeout=30
    BalancerMember http://10.0.0.6:8080 timeout=30
    
    # Mark a backend as failed after X errors within N seconds
    # BalancerMember ... retry=30
</Proxy>

<VirtualHost *:80>
    ProxyTimeout 60
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
</VirtualHost>
```

## Monitoring Timeout Events

```bash
# Check Apache error log for timeout-related errors
sudo grep -i timeout /var/log/apache2/error.log

# Watch live
sudo tail -f /var/log/apache2/error.log | grep -i "timeout\|proxy"

# Common log messages:
# "proxy: error reading status line from remote server"  — ProxyTimeout hit
# "server has not sent any data"                         — backend slow
```

## Recommended Settings by Workload

| Use Case | Timeout | ProxyTimeout |
|---------|---------|-------------|
| REST API (fast) | 30s | 30s |
| Web application | 60s | 60s |
| Report generation | 60s | 300s |
| File upload/download | 120s | 600s |
| Long-poll / SSE | 60s | 3600s |

## Conclusion

Properly tuned timeouts prevent Apache from holding resources on stuck connections while allowing legitimate slow operations to complete. Use `ProxyTimeout` for backend wait time, `RequestReadTimeout` for client-side timeouts, and set per-location overrides for endpoints with unusual latency requirements.
