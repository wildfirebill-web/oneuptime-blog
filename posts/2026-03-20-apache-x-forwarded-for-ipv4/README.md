# How to Configure Apache to Handle X-Forwarded-For Headers with IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Apache, X-Forwarded-For, IPv4, Reverse Proxy, mod_remoteip, Logging

Description: Configure Apache to correctly parse and trust X-Forwarded-For headers from trusted IPv4 reverse proxies using mod_remoteip to log and use real client addresses.

## Introduction

When Apache sits behind a reverse proxy (Nginx, AWS ALB, Cloudflare), the client IP in `REMOTE_ADDR` is the proxy's IP. The `X-Forwarded-For` header contains the real client IPv4. Apache's `mod_remoteip` extracts the real client IP from this header, enabling accurate logging and access control.

## Enabling mod_remoteip

```bash
# Enable the module (Ubuntu/Debian)
sudo a2enmod remoteip
sudo systemctl reload apache2

# On RHEL/CentOS — mod_remoteip is usually included in the base install
# Add to /etc/httpd/conf.modules.d/00-base.conf:
# LoadModule remoteip_module modules/mod_remoteip.so
```

## Basic Configuration

Add to your virtual host or `apache2.conf`:

```apache
# /etc/apache2/conf-available/remoteip.conf

<IfModule mod_remoteip.c>
    # The header containing the real client IP
    RemoteIPHeader X-Forwarded-For
    
    # Trust only these proxy IPs to set the X-Forwarded-For header
    # Only accept the header from known, trusted proxies
    RemoteIPTrustedProxy 10.0.0.0/8
    RemoteIPTrustedProxy 172.16.0.0/12
    RemoteIPTrustedProxy 192.168.0.0/16
    
    # For AWS ALB: trust the ALB subnet
    RemoteIPTrustedProxy 10.0.2.0/24
    
    # For Cloudflare: trust Cloudflare's IP ranges
    # RemoteIPTrustedProxy 103.21.244.0/22
    # RemoteIPTrustedProxy 103.22.200.0/22
    # (add all Cloudflare ranges from cloudflare.com/ips-v4)
</IfModule>
```

Enable and reload:

```bash
sudo a2enconf remoteip
sudo systemctl reload apache2
```

## Using RemoteIPProxyProtocol (HAProxy/Nginx)

If your proxy uses the PROXY protocol instead of X-Forwarded-For:

```apache
<VirtualHost *:80>
    # Enable PROXY protocol on this virtual host
    RemoteIPProxyProtocol On
    RemoteIPProxyProtocolExceptions 127.0.0.1 ::1
</VirtualHost>
```

## Updating the Access Log Format

After enabling `mod_remoteip`, update the `LogFormat` to use `%a` (the remote IP after proxy extraction):

```apache
# /etc/apache2/apache2.conf

# Default combined format uses %h (hostname before remoteip processing)
# Change to %a to log the real client IP
LogFormat "%a %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined

# Note: %h performs DNS lookup, %a is the raw IP (faster)
# The combined format already uses %h; to show real client IP, use:
LogFormat "%{X-Forwarded-For}i %l %u %t \"%r\" %>s %O" xforwardedfor
```

Or create a custom log format:

```apache
LogFormat "%a %{X-Forwarded-For}i %l %u %t \"%r\" %>s %b" combined_proxy
CustomLog /var/log/apache2/access.log combined_proxy
```

## Access Control by Real Client IP

With `mod_remoteip` active, Apache's `Require ip` directive uses the extracted real IP:

```apache
<Location /admin>
    <RequireAll>
        Require method GET POST
        # These CIDR ranges now refer to real client IPs (not proxy IPs)
        Require ip 203.0.113.0/24
        Require ip 10.100.0.0/16
    </RequireAll>
</Location>
```

## Verifying the Configuration

Test that the real IP is being extracted:

```apache
# Add a CGI handler or use mod_status to verify
<Location /test-ip>
    SetHandler server-status
    Require local
</Location>
```

Or create a PHP test:

```php
<?php
// test-remoteip.php
header('Content-Type: text/plain');
echo "REMOTE_ADDR (after mod_remoteip): " . $_SERVER['REMOTE_ADDR'] . "\n";
echo "X-Forwarded-For: " . ($_SERVER['HTTP_X_FORWARDED_FOR'] ?? 'not present') . "\n";
echo "REMOTE_HOST: " . ($_SERVER['REMOTE_HOST'] ?? 'not present') . "\n";
```

Send a test request from behind the proxy and verify `REMOTE_ADDR` shows the client IP, not the proxy IP.

## Security Consideration

Never trust `X-Forwarded-For` from unknown sources. A malicious client can set this header to spoof IP addresses. The `RemoteIPTrustedProxy` directive limits trust to only the specified proxy addresses — only their `X-Forwarded-For` values are processed.

## Conclusion

`mod_remoteip` is the modern, built-in Apache solution for real client IP extraction. Configure it with a strict `RemoteIPTrustedProxy` list, update your log format to use `%a`, and verify access control rules use real client IPs rather than proxy IPs.
