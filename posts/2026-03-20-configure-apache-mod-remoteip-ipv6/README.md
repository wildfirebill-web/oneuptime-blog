# How to Configure Apache mod_remoteip for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Apache, mod_remoteip, X-Forwarded-For, Reverse Proxy

Description: Learn how to configure Apache mod_remoteip to correctly identify IPv6 client addresses when Apache is behind a load balancer or reverse proxy that sends X-Forwarded-For headers.

## Why mod_remoteip for IPv6?

When Apache is behind a load balancer:
- `%h` (REMOTE_ADDR) shows the load balancer's IPv6 address
- The real client IP is in `X-Forwarded-For` header
- `mod_remoteip` overrides REMOTE_ADDR with the real client IP

```
Client (2001:db8::client) → LB (2001:db8::lb) → Apache
Apache sees REMOTE_ADDR = 2001:db8::lb (wrong!)
X-Forwarded-For: 2001:db8::client (real IP)
mod_remoteip corrects this → %a = 2001:db8::client
```

## Enable mod_remoteip

```bash
# Debian/Ubuntu
a2enmod remoteip
systemctl restart apache2

# RHEL/CentOS
# mod_remoteip is included in httpd
```

## Basic mod_remoteip Configuration

```apache
# /etc/apache2/conf-available/remoteip.conf

<IfModule mod_remoteip.c>
    # Header containing the real client IP
    RemoteIPHeader X-Forwarded-For

    # Trust IPv6 load balancer addresses
    RemoteIPTrustedProxy 2001:db8:lb::/64

    # Also trust IPv4 load balancers
    RemoteIPTrustedProxy 192.168.1.0/24

    # Trust a specific IPv6 load balancer
    RemoteIPTrustedProxy 2001:db8::lb1
    RemoteIPTrustedProxy 2001:db8::lb2
</IfModule>
```

```bash
# Enable the configuration
a2enconf remoteip
systemctl reload apache2
```

## Update Log Format to Use Real IP

```apache
# After mod_remoteip, use %a for the real client IP
# %h = hostname (REMOTE_ADDR, already replaced by mod_remoteip)
# %a = real IP address after remoteip processing
# %{c}a = actual connection IP (original REMOTE_ADDR before override)

LogFormat "%a %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined_real
LogFormat "%{c}a %a %l %u %t \"%r\" %>s" combined_debug
# %{c}a = load balancer IP, %a = real client IP

# Apply in virtual host
<VirtualHost *:80>
    CustomLog ${APACHE_LOG_DIR}/access.log combined_real
</VirtualHost>
```

## Use X-Real-IP Instead of X-Forwarded-For

```apache
<IfModule mod_remoteip.c>
    # Some load balancers use X-Real-IP header
    RemoteIPHeader X-Real-IP

    # Trust the IPv6 LB range
    RemoteIPTrustedProxy 2001:db8:lb::/64
</IfModule>
```

## Verify mod_remoteip is Working

```bash
# Check mod_remoteip is loaded
apache2ctl -M | grep remoteip

# Test: check what IP Apache sees
curl -H "X-Forwarded-For: 2001:db8::client" \
    http://localhost/server-info 2>&1 | grep -i 'remote\|real'

# Add a test endpoint to show the remote IP
<Location /myip>
    SetHandler modinfo
</Location>

# Or use a PHP/CGI script that prints REMOTE_ADDR
# After mod_remoteip, REMOTE_ADDR = real client IP
```

## Trusted Proxy vs Internal Proxy

```apache
<IfModule mod_remoteip.c>
    RemoteIPHeader X-Forwarded-For

    # RemoteIPTrustedProxy: Trust these IPs when they appear in X-Forwarded-For chain
    # IPs in the chain that match are removed from the chain
    RemoteIPTrustedProxy 2001:db8:lb::/64

    # RemoteIPInternalProxy: Treat these as internal (don't appear in X-Forwarded-For)
    # Used for intermediate proxies that don't add themselves to X-Forwarded-For
    RemoteIPInternalProxy ::1
    RemoteIPInternalProxy 2001:db8::internal-proxy
</IfModule>
```

## Summary

Configure `mod_remoteip` in Apache with `RemoteIPHeader X-Forwarded-For` and `RemoteIPTrustedProxy 2001:db8:lb::/64` to correctly extract real IPv6 client addresses from X-Forwarded-For headers. After enabling, use `%a` in log format (not `%h`) for the real client IP, and `%{c}a` for the connection IP (load balancer). Enable with `a2enmod remoteip` and apply trusted IPv6 proxy ranges to prevent IP spoofing from untrusted sources.
