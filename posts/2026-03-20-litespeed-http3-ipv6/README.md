# How to Configure LiteSpeed HTTP/3 with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: LiteSpeed, HTTP/3, QUIC, IPv6, Web Server

Description: Configure LiteSpeed Web Server to enable HTTP/3 and QUIC over IPv6 addresses for improved performance and lower latency.

## Overview

LiteSpeed Web Server (LSWS) and OpenLiteSpeed have supported QUIC/HTTP/3 since early versions. Configuration is done through the WebAdmin console or configuration files.

## Installing OpenLiteSpeed

```bash
# Install OpenLiteSpeed on Ubuntu

wget -O - https://rpms.litespeedtech.com/debian/enable_lst_debain_repo.sh | sudo bash
sudo apt-get install openlitespeed

# Start and enable
sudo systemctl enable lsws
sudo systemctl start lsws

# Access WebAdmin at https://YOUR-IP:7080
```

## Step 1: Add an IPv6 Listener in WebAdmin

Navigate to **Configuration → Listeners → Add**:

```text
Listener Name: HTTPS-IPv6
IP Address:    [::] (all IPv6 interfaces)
Port:          443
Secure:        Yes (SSL)
Protocol:      HTTP/3, HTTP/2, HTTP/1.1
```

Or via the configuration file at `/usr/local/lsws/conf/httpd_config.conf`:

```text
listener HTTPS-IPv6 {
  # Bind to all IPv6 addresses
  address               [::]:443
  secure                1

  # Enable QUIC/HTTP3
  quic                  1

  # SSL settings
  keyFile               /etc/ssl/private/example.com.key
  certFile              /etc/ssl/certs/example.com.crt
  certChain             1

  # SSL protocols - TLS 1.3 required for HTTP/3
  sslProtocol           TLS1.3 TLS1.2

  # QUIC options
  quicShmDir            /dev/shm
  quicCertUpdateInterval 300
}
```

## Step 2: Configure QUIC Parameters

```text
# In the listener configuration
quic {
  # Maximum connections
  maxConnections        10000

  # Enable connection migration (important for IPv6 mobility)
  migration             1

  # GSO (Generic Segmentation Offload) for performance
  gso                   1

  # ALPN protocols
  alpn                  h3 h3-29 h2 http/1.1
}
```

## Step 3: Virtual Host Configuration

```text
virtualHost example.com {
  vhRoot                /var/www/example.com/
  configFile            $SERVER_ROOT/conf/vhosts/example.com/vhconf.conf
  allowSymbolLink       1
  enableScript          1
  restrained            1

  # Map listener to virtual host
  # (done in Listeners section)
}
```

## Step 4: Enable Alt-Svc Header

The Alt-Svc header advertises HTTP/3 availability to browsers:

```text
# In virtual host or server-level rewrite rules
rewrite {
  rules                 <<<END_RULES
    RewriteEngine On
    Header always set Alt-Svc 'h3=":443"; ma=86400'
  END_RULES
}
```

## Step 5: Firewall Rules

```bash
# Allow UDP 443 for QUIC/HTTP3
sudo ufw allow 443/udp
sudo ufw allow 443/tcp

# ip6tables rules
sudo ip6tables -A INPUT -p udp --dport 443 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT

# For the WebAdmin console
sudo ip6tables -A INPUT -p tcp --dport 7080 -j ACCEPT
```

## Verification

```bash
# Check OpenLiteSpeed is listening on IPv6 UDP 443
sudo ss -tulnp | grep lshttpd

# Test HTTP/3 over IPv6
curl -6 --http3 https://[2001:db8::1]/ -v 2>&1 | grep -E "QUIC|HTTP/3"

# Check response headers include Alt-Svc
curl -6 -I https://example.com | grep -i alt-svc

# Check LiteSpeed QUIC stats
curl http://localhost:7080/_admin/qperf
```

## Troubleshooting Common Issues

```bash
# Check LiteSpeed error log
tail -f /usr/local/lsws/logs/error.log | grep -i quic

# Verify SSL certificate supports TLS 1.3
openssl s_client -connect [2001:db8::1]:443 -tls1_3

# Check QUIC is actually negotiated (look for ALPN h3)
openssl s_client -alpn h3 -connect [2001:db8::1]:443
```

## Monitoring

Use [OneUptime](https://oneuptime.com) to continuously monitor LiteSpeed availability over IPv6. Configure HTTP monitors that check the Alt-Svc header value and alert if HTTP/3 support is unexpectedly dropped.

## Conclusion

LiteSpeed makes HTTP/3 over IPv6 straightforward through its WebAdmin interface or configuration files. Key steps are: configure an IPv6 listener with QUIC enabled, ensure TLS 1.3, set the Alt-Svc header, and open UDP 443 in your firewall.
