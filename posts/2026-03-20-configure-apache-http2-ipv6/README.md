# How to Configure Apache HTTP/2 with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Apache, HTTP/2, IPv6, Web Server, TLS, Performance, mod_http2

Description: Configure Apache HTTP Server with HTTP/2 protocol support on IPv6 interfaces, enabling modern multiplexed connections for improved web performance.

---

Apache HTTP Server supports HTTP/2 through the `mod_http2` module. Enabling HTTP/2 on IPv6 requires enabling the module, configuring TLS (required for HTTP/2), and ensuring Apache listens on IPv6 interfaces.

## Enabling mod_http2

```bash
# Enable HTTP/2 module
sudo a2enmod http2
sudo a2enmod ssl

# Verify module is loaded
apache2ctl -M | grep "http2"
```

## Apache Configuration with HTTP/2 and IPv6

```apache
# /etc/apache2/ports.conf
# Listen on both IPv4 and IPv6 for HTTP and HTTPS
Listen 80
Listen [::]:80

Listen 443
Listen [::]:443
```

```apache
# /etc/apache2/sites-available/yourdomain.conf

# HTTP VirtualHost (redirect to HTTPS)
<VirtualHost *:80>
    ServerName yourdomain.com

    # Redirect all traffic to HTTPS
    RewriteEngine On
    RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]
</VirtualHost>

# HTTPS VirtualHost with HTTP/2 and IPv6
<VirtualHost *:443>
    ServerName yourdomain.com

    # Enable HTTP/2 (TLS required)
    Protocols h2 h2c http/1.1

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/yourdomain.com/cert.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/yourdomain.com/privkey.pem
    SSLCertificateChainFile /etc/letsencrypt/live/yourdomain.com/chain.pem

    # Modern TLS settings (required for HTTP/2)
    SSLProtocol -all +TLSv1.2 +TLSv1.3
    SSLHonorCipherOrder off
    SSLSessionTickets off

    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Server push via Link header
    <FilesMatch "\.html$">
        Header add Link "</css/styles.css>; rel=preload; as=style"
        Header add Link "</js/app.js>; rel=preload; as=script"
    </FilesMatch>

    ErrorLog ${APACHE_LOG_DIR}/yourdomain_error.log
    CustomLog ${APACHE_LOG_DIR}/yourdomain_access.log combined
</VirtualHost>
```

## IPv6-Only Apache Configuration

```apache
# Listen only on IPv6
<VirtualHost [2001:db8::1]:443>
    ServerName yourdomain.com

    Protocols h2 http/1.1

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/server.crt
    SSLCertificateKeyFile /etc/ssl/private/server.key

    DocumentRoot /var/www/html
</VirtualHost>
```

## mod_http2 Tuning

```apache
# /etc/apache2/conf-available/http2.conf

# Global HTTP/2 settings
H2Direct on

# Session cache (improves 0-RTT)
H2SessionExtraFiles 5

# Stream push configuration
H2Push on
H2PushPriority * after 32

# Worker thread settings
H2MinWorkers 10
H2MaxWorkers 75

# Timeout settings
H2Timeout 5
H2KeepAliveTimeout 5

# Header table size for HPACK compression
H2TLSWarmUpSize 0
```

## Enabling and Testing

```bash
# Enable site and HTTP/2 config
sudo a2ensite yourdomain
sudo a2enconf http2

# Test configuration
sudo apache2ctl configtest

# Restart Apache
sudo systemctl restart apache2

# Verify HTTP/2 is active
curl -6 --http2 -I https://yourdomain.com/
# Look for: HTTP/2 200

# Check which protocols Apache is advertising
openssl s_client \
  -connect '[2001:db8::1]':443 \
  -servername yourdomain.com \
  -alpn h2 < /dev/null 2>&1 | grep "Protocol"
# Expected: Protocol : TLSv1.3 or similar, with ALPN negotiated h2
```

## Monitoring HTTP/2 on Apache

```apache
# Enable mod_status for protocol statistics
<Location /server-status>
    SetHandler server-status
    Require ip ::1 127.0.0.1
</Location>

# Log HTTP/2 protocol version
LogFormat "%h %l %u %t \"%r\" %>s %b %{ALPN}e" http2_combined
CustomLog /var/log/apache2/http2_access.log http2_combined
```

Apache's `mod_http2` with IPv6-capable virtual hosts provides a reliable path to HTTP/2 multiplexing for websites served from IPv6 infrastructure, improving performance through connection multiplexing and header compression.
