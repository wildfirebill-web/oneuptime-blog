# How to Disable IPv6 in Apache and Listen Only on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Apache, IPv4, IPv6, Listen Directive, Networking, Configuration

Description: Configure Apache HTTP Server to disable IPv6 listening and bind exclusively to IPv4 addresses, ensuring consistent behavior on IPv4-only networks.

## Introduction

Apache listens on IPv6 by default (`:::80`) on dual-stack systems, which on Linux also accepts IPv4 connections (due to IPV6_V6ONLY being off). Explicitly binding to IPv4 only improves clarity, avoids confusion, and is required on IPv4-only infrastructure.

## Understanding Default Apache Behavior

On a typical Linux system with Apache default config:

```bash
# Default: Apache listens on all IPv6 addresses (which includes IPv4 via dual-stack)
sudo ss -tlnp | grep apache2
# LISTEN 0 511 :::80 :::*  users:(("apache2",...))
# This :::80 accepts BOTH IPv4 and IPv6 connections on most Linux systems
```

## Disabling IPv6 and Binding IPv4 Only

Edit `ports.conf` to replace IPv6 wildcard with IPv4:

```apache
# /etc/apache2/ports.conf (Ubuntu/Debian)

# Remove the default IPv6 wildcard:
# Listen 80  ← this becomes :::80 on dual-stack systems

# Replace with explicit IPv4 wildcard:
Listen 0.0.0.0:80
Listen 0.0.0.0:443
```

Update virtual host definitions to use IPv4:

```apache
# /etc/apache2/sites-available/000-default.conf

# Old: <VirtualHost *:80>
# New: bind explicitly to IPv4 wildcard
<VirtualHost 0.0.0.0:80>
    ServerName example.com
    DocumentRoot /var/www/html
</VirtualHost>
```

## Disabling IPv6 at the OS Level (Optional)

For complete IPv6 removal at the kernel level:

```bash
# /etc/sysctl.conf or /etc/sysctl.d/99-disable-ipv6.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

Apply the changes:

```bash
sudo sysctl -p /etc/sysctl.d/99-disable-ipv6.conf
```

## Verifying IPv6 Is Not Listening

```bash
# After restarting Apache, verify only IPv4 is listening
sudo systemctl restart apache2
sudo ss -tlnp | grep apache2

# Desired output (IPv4 only):
# LISTEN 0 511 0.0.0.0:80 0.0.0.0:*  users:(("apache2",...))

# Confirm no IPv6 listener
sudo ss -tlnp | grep ':::80'
# Should return no output
```

## Using Apache's AddressFamily Directive (MPM Worker/Event)

Some configurations support the `AddressFamily` directive in virtual hosts:

```apache
# This is NOT a standard Apache directive but available in some distros
# The recommended way is always via Listen directive

<VirtualHost 0.0.0.0:80>
    ServerName example.com
    DocumentRoot /var/www/html
</VirtualHost>
```

## Testing IPv4-Only Access

```bash
# Test IPv4 access (should work)
curl -4 http://example.com

# Test IPv6 access (should fail or time out)
curl -6 http://example.com
# Expected: curl: (7) Failed to connect to example.com: Connection refused
```

## Conclusion

To make Apache IPv4-only, change `Listen 80` to `Listen 0.0.0.0:80` in `ports.conf` and update VirtualHost addresses to match. For total IPv6 isolation, also disable IPv6 at the kernel level with sysctl. Always validate with `ss -tlnp` after restarting to confirm no `:::80` listeners remain.
