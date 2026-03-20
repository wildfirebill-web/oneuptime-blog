# How to Configure Apache to Listen on IPv6 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Apache, Web Server, Listen Directive, Network Configuration

Description: Learn how to configure Apache HTTP Server to listen on IPv6 addresses using the Listen directive, including listening on all interfaces, specific addresses, and dual-stack configuration.

## Basic IPv6 Listen Directives

```apache
# In /etc/apache2/ports.conf or httpd.conf

# Listen on all IPv6 addresses on port 80

Listen [::]:80

# Listen on a specific IPv6 address
Listen [2001:db8::10]:80

# Listen on IPv6 loopback
Listen [::1]:80

# Listen on all IPv4 and IPv6 addresses
Listen 80        # IPv4: 0.0.0.0:80
Listen [::]:80   # IPv6: all interfaces
```

## Apache IPv6 Configuration File

```apache
# /etc/apache2/ports.conf

# Listen on both IPv4 and IPv6 port 80
Listen 80
Listen [::]:80

# HTTPS
<IfModule ssl_module>
    Listen 443
    Listen [::]:443
</IfModule>

<IfModule mod_gnutls.c>
    Listen 443
    Listen [::]:443
</IfModule>
```

## Dual-Stack vs IPv6-Only

```apache
# Dual-stack: respond to both IPv4 and IPv6
Listen 0.0.0.0:80
Listen [::]:80

# IPv6-only server
Listen [::]:80
# (Don't add Listen 80 or Listen 0.0.0.0:80)

# Check if Apache is listening
# ss -tlnp | grep apache
# netstat -tlnp | grep apache
```

## VirtualHost with IPv6

```apache
# IPv6-only virtual host
<VirtualHost [::]:80>
    ServerName example.com
    DocumentRoot /var/www/html/example
    ErrorLog ${APACHE_LOG_DIR}/example-error.log
    CustomLog ${APACHE_LOG_DIR}/example-access.log combined
</VirtualHost>

# Dual-stack virtual host
<VirtualHost *:80>
    # * matches both IPv4 and IPv6 Listen directives
    ServerName example.com
    DocumentRoot /var/www/html/example
</VirtualHost>
```

## Enable IPv6 Module

```bash
# Check if IPv6 is supported
apache2 -V | grep IPv6
# or
httpd -V | grep IPv6
# Should show: "IPv6 Listening" or "IPv6 support" in output

# On Debian/Ubuntu - enable IPv6 module is usually built-in
# On RHEL/CentOS - also usually built-in
# Verify via configuration
grep -r 'IPv6\|Listen.*\[:' /etc/apache2/
```

## Apply and Verify

```bash
# Test configuration syntax
apache2ctl configtest
# or
httpd -t

# Restart Apache
systemctl restart apache2
# or
systemctl restart httpd

# Verify IPv6 listening
ss -6 -tlnp | grep apache
# Expected: tcp6  LISTEN  0  128  [::]:80  [::]:*  apache2

# Test with curl
curl -6 http://[::1]/
curl -6 http://[2001:db8::10]/
```

## Summary

Configure Apache to listen on IPv6 by adding `Listen [::]:80` to `/etc/apache2/ports.conf`. For dual-stack, include both `Listen 80` (IPv4) and `Listen [::]:80` (IPv6). Use `<VirtualHost [::]:80>` for IPv6-only virtual hosts or `<VirtualHost *:80>` to match both. Test syntax with `apache2ctl configtest` and verify with `ss -6 -tlnp | grep apache`.
