# How to Configure Apache with IPv6 Virtual Hosts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Apache, VirtualHost, Web Server, Name-Based Hosting

Description: A practical guide to setting up multiple Apache virtual hosts served over IPv6, covering name-based hosting, IP-based hosting, and mixed IPv4/IPv6 scenarios.

## IPv6 Virtual Host Fundamentals

```apache
# In Apache, IPv6 addresses in VirtualHost must be in brackets

# <VirtualHost [IPv6-address]:port>

# All three syntax forms:
<VirtualHost [::]:80>           # All IPv6 interfaces
<VirtualHost [2001:db8::10]:80>  # Specific IPv6 address
<VirtualHost *:80>               # All interfaces (IPv4 + IPv6)
```

## Name-Based IPv6 Virtual Hosting

```apache
# /etc/apache2/ports.conf
Listen [::]:80

# /etc/apache2/sites-available/ipv6-sites.conf

# Site 1
<VirtualHost [::]:80>
    ServerName site1.example.com
    ServerAlias www.site1.example.com
    DocumentRoot /var/www/site1

    ErrorLog  ${APACHE_LOG_DIR}/site1-error.log
    CustomLog ${APACHE_LOG_DIR}/site1-access.log combined
</VirtualHost>

# Site 2
<VirtualHost [::]:80>
    ServerName site2.example.com
    DocumentRoot /var/www/site2

    ErrorLog  ${APACHE_LOG_DIR}/site2-error.log
    CustomLog ${APACHE_LOG_DIR}/site2-access.log combined
</VirtualHost>

# Default virtual host
<VirtualHost [::]:80>
    ServerName default.example.com
    DocumentRoot /var/www/default
</VirtualHost>
```

## IP-Based IPv6 Virtual Hosting

```apache
# Each site has its own IPv6 address
# Requires multiple IPv6 addresses on the server

# /etc/apache2/ports.conf
Listen [2001:db8::10]:80
Listen [2001:db8::20]:80

# Site 1 on first IPv6 address
<VirtualHost [2001:db8::10]:80>
    ServerName site1.example.com
    DocumentRoot /var/www/site1
</VirtualHost>

# Site 2 on second IPv6 address
<VirtualHost [2001:db8::20]:80>
    ServerName site2.example.com
    DocumentRoot /var/www/site2
</VirtualHost>
```

## Mixed IPv4 and IPv6 Virtual Hosts

```apache
# /etc/apache2/ports.conf
Listen 80
Listen [::]:80

# Site accessible via both IPv4 and IPv6
# Using wildcard matches both Listen directives
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/example
</VirtualHost>

# Another site
<VirtualHost *:80>
    ServerName other.example.com
    DocumentRoot /var/www/other
</VirtualHost>
```

## HTTPS IPv6 Virtual Hosts

```apache
# /etc/apache2/ports.conf
Listen 443
Listen [::]:443

# Redirect HTTP to HTTPS
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
</VirtualHost>

# HTTPS virtual host
<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

    DocumentRoot /var/www/example

    # Security headers
    Header always set X-Content-Type-Options nosniff
    Header always set Strict-Transport-Security "max-age=31536000"
</VirtualHost>
```

## Enable and Test

```bash
# Enable site configuration
a2ensite ipv6-sites.conf

# Test syntax
apache2ctl configtest

# Reload Apache
systemctl reload apache2

# View virtual host layout
apache2ctl -S

# Test name-based IPv6 virtual hosts
curl -6 -H "Host: site1.example.com" http://[::1]/
curl -6 -H "Host: site2.example.com" http://[::1]/

# Test with real IPv6 address
curl -6 http://[2001:db8::10]/ -H "Host: site1.example.com"
```

## Summary

Configure Apache IPv6 virtual hosts with `<VirtualHost [::]:80>` for all-IPv6 interfaces or `<VirtualHost [2001:db8::10]:80>` for a specific address. Name-based hosting over IPv6 works the same as IPv4 - Apache matches `Host:` header to `ServerName`. Use `<VirtualHost *:80>` to match all Listen directives (both IPv4 and IPv6). Verify layout with `apache2ctl -S` and test with `curl -6 -H "Host: example.com"`.
