# How to Configure Apache VirtualHosts with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Apache, VirtualHost, Web Server, Network Configuration

Description: Learn how to configure Apache VirtualHost directives for IPv6 addresses, including IPv6-specific virtual hosts, name-based virtual hosting over IPv6, and dual-stack configurations.

## IPv6 VirtualHost Syntax

```apache
# IPv6 addresses in VirtualHost directives must be in brackets

# IPv6-only virtual host
<VirtualHost [2001:db8::10]:80>
    ServerName example.com
    DocumentRoot /var/www/example
</VirtualHost>

# All IPv6 interfaces
<VirtualHost [::]:80>
    ServerName default.example.com
    DocumentRoot /var/www/default
</VirtualHost>
```

## Name-Based Virtual Hosting over IPv6

```apache
# /etc/apache2/ports.conf
Listen [::]:80
Listen 80

# /etc/apache2/sites-available/ipv6-vhosts.conf

# Default/catch-all for all IPv4 and IPv6 requests
<VirtualHost *:80>
    ServerName default.example.com
    DocumentRoot /var/www/default
</VirtualHost>

# Specific site for IPv6 clients
<VirtualHost [::]:80>
    ServerName www.example.com
    ServerAlias example.com
    DocumentRoot /var/www/example
    ErrorLog ${APACHE_LOG_DIR}/example-error.log
    CustomLog ${APACHE_LOG_DIR}/example-access.log combined
</VirtualHost>
```

## Dual-Stack VirtualHost

```apache
# Two separate VirtualHost blocks for IPv4 and IPv6

# IPv4 virtual host
<VirtualHost 192.168.1.10:80>
    ServerName example.com
    DocumentRoot /var/www/example
</VirtualHost>

# IPv6 virtual host (same DocumentRoot)
<VirtualHost [2001:db8::10]:80>
    ServerName example.com
    DocumentRoot /var/www/example
</VirtualHost>
```

## Multiple Sites on a Single IPv6 Address

```apache
# Name-based virtual hosting (most common for multiple sites)
<VirtualHost [::]:80>
    ServerName site1.example.com
    DocumentRoot /var/www/site1
</VirtualHost>

<VirtualHost [::]:80>
    ServerName site2.example.com
    DocumentRoot /var/www/site2
</VirtualHost>

<VirtualHost [::]:80>
    ServerName site3.example.com
    DocumentRoot /var/www/site3
</VirtualHost>
```

## IPv6 HTTPS VirtualHost

```apache
<VirtualHost [::]:443>
    ServerName secure.example.com

    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/example.crt
    SSLCertificateKeyFile   /etc/ssl/private/example.key
    SSLProtocol             all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite          ECDHE-ECDSA-AES256-GCM-SHA384

    DocumentRoot /var/www/secure

    ErrorLog  ${APACHE_LOG_DIR}/secure-error.log
    CustomLog ${APACHE_LOG_DIR}/secure-access.log combined

    Header always set Strict-Transport-Security "max-age=31536000"
</VirtualHost>
```

## Enable and Test VirtualHost

```bash
# Enable the site (Debian/Ubuntu)
a2ensite ipv6-vhosts.conf
systemctl reload apache2

# Test configuration
apache2ctl -S
# Shows list of virtual hosts with their addresses

# Test IPv6 virtual host
curl -6 -H "Host: www.example.com" http://[::1]/
curl -6 http://[2001:db8::10]/

# Check which virtual host is serving a request
curl -6 -v http://[2001:db8::10]/ 2>&1 | grep Server
```

## Summary

Configure Apache IPv6 VirtualHosts with `<VirtualHost [::]:80>` (all IPv6) or `<VirtualHost [2001:db8::10]:80>` (specific address). IPv6 addresses must be in brackets. Use `<VirtualHost *:80>` to match both IPv4 and IPv6 Listen directives. For dual-stack, create two VirtualHost blocks — one for IPv4 and one for IPv6 — pointing to the same DocumentRoot. Verify with `apache2ctl -S` and test with `curl -6`.
