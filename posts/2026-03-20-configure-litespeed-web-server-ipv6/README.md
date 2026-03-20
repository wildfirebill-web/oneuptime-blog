# How to Configure LiteSpeed Web Server with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: LiteSpeed, IPv6, Web Server, HTTP/3, QUIC, Performance, Admin Console

Description: Configure LiteSpeed Web Server to accept connections over IPv6, enable HTTP/3, and manage virtual hosts with IPv6 listener addresses for high-performance web serving.

---

LiteSpeed Web Server supports IPv6 and HTTP/3 out of the box. Configuring it for IPv6 involves setting up listeners through the LiteSpeed WebAdmin console or directly in configuration files.

## Installing LiteSpeed Web Server

```bash
# OpenLiteSpeed (open source version)

sudo apt install wget curl -y

# Add OpenLiteSpeed repository
wget -O - https://rpms.litespeedtech.com/debian/enable_lst_debian_repo.sh | bash
sudo apt update && sudo apt install openlitespeed -y

# Start LiteSpeed
sudo systemctl enable --now lshttpd

# Access WebAdmin console
# Default: https://server-ip:7080
# Default credentials: admin/123456 (change immediately)
```

## Configuring IPv6 Listener via WebAdmin

Navigate to WebAdmin Console → Configuration → Server → Listeners:

```bash
# WebAdmin URL over IPv6
https://[2001:db8::1]:7080/

# Create a new listener:
# Listener Name: IPv6-HTTPS
# IP Address: [2001:db8::1] or ANY (for all interfaces)
# Port: 443
# Secure: Yes (TLS)
# IPv6 Only: No (dual-stack) or Yes (IPv6-only)
```

## Direct Configuration File Approach

```xml
<!-- /usr/local/lsws/conf/httpd_config.conf -->

<listener IPv6HTTPS>
  address                 [::]:443
  secure                  1
  keyFile                 /etc/letsencrypt/live/yourdomain.com/privkey.pem
  certFile                /etc/letsencrypt/live/yourdomain.com/fullchain.pem
  certChain               1
  ciphers                 ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256
  enableSpdy              15  # SPDY, HTTP/2, HTTP/3 flags
  sslProtocol             24  # TLS 1.2 + 1.3
</listener>

<!-- Map listener to virtual host -->
<listenerVhost>
  <vhostMap>
    <vhost>         yourVirtualHost </vhost>
    <domains>       yourdomain.com </domains>
  </vhostMap>
</listenerVhost>
```

## Enabling HTTP/3 (QUIC) on IPv6

LiteSpeed has excellent HTTP/3 support. Enable it in the WebAdmin:

```bash
# In WebAdmin Console:
# Configuration → Server → General → HTTP/3 (QUIC):
# Enable: Yes

# Or in configuration file:
# quicEnable = 1
# quicShmDir = /tmp/lshttpd/quic
```

The Alt-Svc header is automatically added when HTTP/3 is enabled.

## Virtual Host Configuration for IPv6

```xml
<!-- /usr/local/lsws/conf/vhosts/yourdomain/vhconf.conf -->

<virtualHost>
  vhRoot                  /var/www/yourdomain
  configFile              $SERVER_ROOT/conf/vhosts/yourdomain/vhconf.conf
  allowSymbolLink         1
  enableScript            1
  restrained              0
</virtualHost>

<vhTemplate>
  <!-- Virtual host accessible over IPv6 via the listener -->
</vhTemplate>
```

## Testing LiteSpeed over IPv6

```bash
# Test HTTP/1.1 over IPv6
curl -6 http://yourdomain.com/

# Test HTTPS over IPv6
curl -6 https://yourdomain.com/ -v 2>&1 | grep "HTTP/"

# Test HTTP/3 over IPv6
curl --http3 -6 https://yourdomain.com/ -v 2>&1 | grep "HTTP/3"

# Check Alt-Svc header for HTTP/3 advertisement
curl -6 -I https://yourdomain.com/ | grep "alt-svc"

# Verify IPv6 connectivity
ss -tlnp | grep lshttpd
# Should show [::]:443 and [::]:80
```

## LiteSpeed PHP with IPv6

```bash
# LiteSpeed LSPHP listener on IPv6
# /usr/local/lsws/conf/httpd_config.conf

<extprocessor PHPProcess>
  type                    lsapi
  address                 uds://tmp/lshttpd/lsphp.sock
  maxConns                35
  env                     PHP_LSAPI_CHILDREN=35
  initTimeout             60
  retryTimeout            0
  persistConn             1
</extprocessor>
```

## Monitoring LiteSpeed with IPv6

```bash
# LiteSpeed real-time statistics (WebAdmin)
curl -6 https://[2001:db8::1]:7080/ExtApp/PHP_LSAPI_CHILDREN

# Check access logs for protocol info
tail -f /usr/local/lsws/logs/access.log | grep "HTTP/[23]"

# LiteSpeed server status
curl -6 http://[::1]:7080/server-status
```

LiteSpeed's native HTTP/3 implementation and IPv6 listener support make it an excellent choice for high-performance web serving on modern IPv6 infrastructure, with significantly lower resource usage than Apache for the same workload.
