# How to Configure Apache Reverse Proxy with IPv6 Backends

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Apache, Reverse Proxy, mod_proxy, Backend

Description: Learn how to configure Apache as a reverse proxy forwarding requests to IPv6 backend servers using mod_proxy, including HTTP and HTTPS backends.

## Enable Required Modules

```bash
# Enable mod_proxy and related modules

a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests headers
systemctl restart apache2
```

## Basic IPv6 Reverse Proxy

```apache
<VirtualHost *:80>
    ServerName example.com

    # Enable proxy
    ProxyPreserveHost On
    ProxyRequests Off

    # Proxy to IPv6 backend (brackets required)
    ProxyPass        / http://[2001:db8::10]:8080/
    ProxyPassReverse / http://[2001:db8::10]:8080/

    # Pass client IP to backend
    RequestHeader set X-Real-IP "%{REMOTE_ADDR}e"
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}e"
    RequestHeader set X-Forwarded-Proto "http"
</VirtualHost>
```

## HTTPS Frontend to HTTP IPv6 Backend

```apache
<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/example.crt
    SSLCertificateKeyFile /etc/ssl/private/example.key

    ProxyPreserveHost On
    ProxyRequests Off

    ProxyPass        / http://[2001:db8::10]:8080/
    ProxyPassReverse / http://[2001:db8::10]:8080/

    # Tell backend the connection was HTTPS
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "443"
    RequestHeader set X-Real-IP "%{REMOTE_ADDR}e"
</VirtualHost>
```

## Load Balancing with IPv6 Backends

```apache
<Proxy "balancer://ipv6cluster">
    # IPv6 backends - brackets required
    BalancerMember http://[2001:db8::10]:8080 loadfactor=1
    BalancerMember http://[2001:db8::11]:8080 loadfactor=1
    BalancerMember http://[2001:db8::12]:8080 loadfactor=1

    # Standby member
    BalancerMember http://[2001:db8::13]:8080 status=+H

    ProxySet lbmethod=byrequests
</Proxy>

<VirtualHost *:80>
    ServerName example.com

    ProxyPass        / balancer://ipv6cluster/
    ProxyPassReverse / balancer://ipv6cluster/

    ProxyPreserveHost On
</VirtualHost>
```

## Per-Path Proxying

```apache
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/static

    # Proxy /api/ to IPv6 backend
    ProxyPass        /api/ http://[2001:db8::api]:3000/
    ProxyPassReverse /api/ http://[2001:db8::api]:3000/

    # Proxy /ws/ for WebSocket
    ProxyPass        /ws/  ws://[2001:db8::ws]:8080/
    ProxyPassReverse /ws/  ws://[2001:db8::ws]:8080/

    # Serve static files locally
    <Directory /var/www/static>
        Require all granted
    </Directory>
</VirtualHost>
```

## Test Reverse Proxy

```bash
# Test proxy to IPv6 backend
curl -v http://example.com/api/health

# Test with IPv6 client
curl -6 http://[2001:db8::10]/

# Check proxy headers are forwarded
curl -v http://example.com/ 2>&1 | grep -i 'x-forwarded\|x-real'

# Check backend is responding
curl http://[2001:db8::10]:8080/health

# View proxy errors
tail -f /var/log/apache2/error.log | grep proxy
```

## Summary

Configure Apache reverse proxy to IPv6 backends with `ProxyPass / http://[2001:db8::10]:8080/` - brackets are required around IPv6 addresses. Enable `mod_proxy` and `mod_proxy_http` with `a2enmod`. Use `RequestHeader set X-Real-IP "%{REMOTE_ADDR}e"` to pass client IP to backend. For load balancing, use `<Proxy "balancer://name">` with multiple `BalancerMember http://[2001:db8::N]:PORT` entries.
