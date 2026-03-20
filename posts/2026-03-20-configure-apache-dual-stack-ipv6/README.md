# How to Configure Apache Dual-Stack (IPv4 and IPv6)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Apache, Dual-Stack, Web Server, Network Configuration

Description: Learn how to configure Apache HTTP Server to simultaneously serve both IPv4 and IPv6 clients (dual-stack mode) with proper VirtualHost configuration.

## ports.conf for Dual-Stack

```apache
# /etc/apache2/ports.conf

# Listen on IPv4

Listen 0.0.0.0:80

# Listen on IPv6
Listen [::]:80

# HTTPS dual-stack
<IfModule ssl_module>
    Listen 0.0.0.0:443
    Listen [::]:443
</IfModule>
```

## Dual-Stack VirtualHost with Wildcard

```apache
# Using * matches ALL Listen addresses (both IPv4 and IPv6)
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example

    # Logs will show both IPv4 and IPv6 client addresses
    ErrorLog  ${APACHE_LOG_DIR}/example-error.log
    CustomLog ${APACHE_LOG_DIR}/example-access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/example.crt
    SSLCertificateKeyFile /etc/ssl/private/example.key

    DocumentRoot /var/www/example
</VirtualHost>
```

## Explicit Dual-Stack VirtualHosts

```apache
# For sites needing separate control per address family

# IPv4 site
<VirtualHost 192.168.1.10:80>
    ServerName example.com
    DocumentRoot /var/www/example
    # IPv4-specific settings
</VirtualHost>

# IPv6 site (same DocumentRoot and config)
<VirtualHost [2001:db8::10]:80>
    ServerName example.com
    DocumentRoot /var/www/example
    # IPv6-specific settings (can be different)
</VirtualHost>
```

## Detecting IPv6 Clients in Apache

```apache
# Use SetEnvIf to detect IPv6 connections
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/example

    # Set variable for IPv6 clients (address contains colons)
    SetEnvIf Remote_Addr ":" IS_IPV6

    # Different log for IPv4 vs IPv6
    CustomLog ${APACHE_LOG_DIR}/ipv4-access.log combined env=!IS_IPV6
    CustomLog ${APACHE_LOG_DIR}/ipv6-access.log combined env=IS_IPV6
</VirtualHost>
```

## Test Dual-Stack

```bash
# Verify Apache listens on both
ss -tlnp | grep apache
# Should show:
# tcp  LISTEN  0  128  0.0.0.0:80  0.0.0.0:*  users:(("apache2"...))
# tcp  LISTEN  0  128  [::]:80     [::]:*      users:(("apache2"...))

# Test IPv4 access
curl -4 http://example.com

# Test IPv6 access
curl -6 http://example.com

# Test directly by address
curl http://192.168.1.10/ -H "Host: example.com"
curl -6 http://[2001:db8::10]/ -H "Host: example.com"

# Check server-status for both connection types
curl -6 http://[::1]/server-status
```

## Apache Log Format with IP Version

```apache
# Custom log format indicating IP version
LogFormat "%{REMOTE_ADDR}e %h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined_v6
# $h shows the client IP (IPv4 or IPv6 as appropriate)
```

## Summary

Configure Apache dual-stack by adding `Listen 80` and `Listen [::]:80` to `ports.conf`. Use `<VirtualHost *:80>` to match all Listen addresses (both IPv4 and IPv6) in a single block, or use explicit `<VirtualHost 192.168.1.10:80>` and `<VirtualHost [2001:db8::10]:80>` pairs. Verify with `ss -tlnp | grep apache` and test with `curl -4` and `curl -6`. Use `SetEnvIf Remote_Addr ":"` to detect and differentiate IPv6 clients.
