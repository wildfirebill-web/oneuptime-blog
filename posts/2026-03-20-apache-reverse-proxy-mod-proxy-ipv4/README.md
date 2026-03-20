# How to Set Up Apache as a Reverse Proxy Using mod_proxy with IPv4 Backends

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Apache, mod_proxy, Reverse Proxy, IPv4, HTTP, Backend

Description: Configure Apache mod_proxy to forward HTTP requests to IPv4 backend servers, enabling load balancing, SSL termination, and application isolation.

## Introduction

Apache's `mod_proxy` and `mod_proxy_http` modules turn Apache into a powerful reverse proxy. Incoming requests are forwarded to IPv4 backend application servers, while Apache handles SSL, compression, and access control.

## Enabling Required Modules

```bash
# Enable proxy modules
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
sudo a2enmod headers
sudo a2enmod rewrite

# Verify modules are loaded
apache2ctl -M | grep proxy
```

## Basic Reverse Proxy Configuration

```apache
# /etc/apache2/sites-available/reverse-proxy.conf

<VirtualHost *:80>
    ServerName app.example.com

    # Disable forward proxy (security: prevent using Apache as open proxy)
    ProxyRequests Off

    # Preserve original Host header
    ProxyPreserveHost On

    # Forward all requests to the IPv4 backend
    ProxyPass        / http://192.168.1.10:8080/
    ProxyPassReverse / http://192.168.1.10:8080/

    # Pass real client IP to backend
    RequestHeader set X-Real-IP %{REMOTE_ADDR}s
    RequestHeader set X-Forwarded-For %{X-Forwarded-For}e
    RequestHeader set X-Forwarded-Proto http

    ErrorLog  /var/log/apache2/app-error.log
    CustomLog /var/log/apache2/app-access.log combined
</VirtualHost>
```

## Proxying Specific URL Paths

Forward only certain paths to the backend:

```apache
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/html  # Serve static files locally

    # Forward /api/ requests to backend
    ProxyPass        /api/ http://192.168.1.10:8080/api/
    ProxyPassReverse /api/ http://192.168.1.10:8080/api/

    # Forward /app/ to a different backend
    ProxyPass        /app/ http://192.168.1.11:8080/
    ProxyPassReverse /app/ http://192.168.1.11:8080/

    # Everything else is served as static content from DocumentRoot
</VirtualHost>
```

## SSL Termination with Reverse Proxy

Apache terminates SSL and proxies plain HTTP to the backend:

```apache
<VirtualHost *:443>
    ServerName app.example.com

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/app.example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/app.example.com/privkey.pem

    ProxyRequests Off
    ProxyPreserveHost On

    # Inform backend that original connection was HTTPS
    RequestHeader set X-Forwarded-Proto https
    RequestHeader set X-Forwarded-Port 443

    ProxyPass        / http://192.168.1.10:8080/
    ProxyPassReverse / http://192.168.1.10:8080/
</VirtualHost>
```

## Configuring Proxy Timeouts and Connection Settings

```apache
<VirtualHost *:80>
    ServerName api.example.com

    # Connection timeout to backend (default 300s)
    ProxyTimeout 60

    <Proxy http://192.168.1.10:8080/>
        # Connection pool settings
        ProxySet connectiontimeout=10 timeout=60 retry=5
        ProxySet keepalive=On
    </Proxy>

    ProxyPass        / http://192.168.1.10:8080/
    ProxyPassReverse / http://192.168.1.10:8080/
</VirtualHost>
```

## Enabling mod_proxy for HTTPS Backends

For backends serving HTTPS:

```bash
sudo a2enmod proxy_connect
sudo a2enmod ssl
```

```apache
# Proxy to HTTPS backend (SSL passthrough)
SSLProxyEngine On
SSLProxyVerify none   # For self-signed backend certs (disable in production)
ProxyPass        /    https://192.168.1.10:8443/
ProxyPassReverse /    https://192.168.1.10:8443/
```

## Testing the Reverse Proxy

```bash
# Verify proxy is working
curl -v http://app.example.com/api/health

# Check headers being forwarded to backend
curl -v http://app.example.com/ 2>&1 | grep -i "x-forwarded\|x-real"

# Test proxy config syntax
sudo apache2ctl configtest
```

## Conclusion

Apache mod_proxy makes reverse proxying straightforward with `ProxyPass` and `ProxyPassReverse` directives. Always disable forward proxying with `ProxyRequests Off`, enable `ProxyPreserveHost On` for correct virtual host routing on backends, and set `X-Forwarded-Proto` when terminating SSL. Use `ProxyTimeout` and connection pool settings to match your backend's performance profile.
