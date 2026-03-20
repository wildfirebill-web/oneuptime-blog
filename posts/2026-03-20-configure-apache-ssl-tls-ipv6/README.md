# How to Configure Apache SSL/TLS with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Apache, SSL, TLS, HTTPS

Description: Learn how to configure Apache HTTP Server with SSL/TLS encryption for IPv6 listeners, including enabling mod_ssl, creating HTTPS virtual hosts, and configuring modern TLS settings.

## Enable mod_ssl and mod_headers

```bash
# Debian/Ubuntu
a2enmod ssl headers
systemctl restart apache2

# RHEL/CentOS
# mod_ssl is usually pre-installed
yum install mod_ssl
systemctl restart httpd
```

## Basic IPv6 HTTPS Configuration

```apache
# /etc/apache2/ports.conf
Listen 443
Listen [::]:443

# /etc/apache2/sites-available/example-ssl.conf

# HTTP to HTTPS redirect (dual-stack)
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
</VirtualHost>

# HTTPS server (dual-stack, both IPv4 and IPv6)
<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/example.com.crt
    SSLCertificateKeyFile   /etc/ssl/private/example.com.key
    SSLCertificateChainFile /etc/ssl/certs/ca-chain.crt

    # Modern TLS settings
    SSLProtocol             all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite          ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    SSLHonorCipherOrder     off
    SSLSessionTickets       off

    # Security headers
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    Header always set X-Content-Type-Options "nosniff"

    DocumentRoot /var/www/example
    ErrorLog  ${APACHE_LOG_DIR}/example-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/example-ssl-access.log combined
</VirtualHost>
```

## IPv6-Specific HTTPS VirtualHost

```apache
# HTTPS only on IPv6
<VirtualHost [::]:443>
    ServerName ipv6.example.com

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/ipv6.example.com.crt
    SSLCertificateKeyFile /etc/ssl/private/ipv6.example.com.key
    SSLProtocol           all -SSLv3 -TLSv1 -TLSv1.1

    DocumentRoot /var/www/ipv6
</VirtualHost>
```

## Let's Encrypt with Apache IPv6

```bash
# Install certbot for Apache
apt install certbot python3-certbot-apache

# Obtain certificate (works for IPv6-accessible domains)
certbot --apache -d example.com

# If server is IPv6-only, ensure ACME challenge works over IPv6
# Test: can Let's Encrypt reach your server?
curl -6 http://example.com/.well-known/acme-challenge/test
```

## Test HTTPS over IPv6

```bash
# Test HTTPS connection over IPv6
curl -6 -v https://example.com 2>&1 | grep -E 'Connected|TLS|SSL|subject'

# Test with specific IPv6 address
curl --connect-to example.com:443:[2001:db8::10]:443 https://example.com

# OpenSSL test
openssl s_client -connect '[2001:db8::10]:443' -servername example.com

# Check certificate
echo "" | openssl s_client -6 -connect example.com:443 2>&1 | \
    openssl x509 -noout -text | grep -A 2 "Subject:"

# Test TLS version
curl -6 --tlsv1.3 https://example.com
```

## Summary

Configure Apache HTTPS for IPv6 by adding `Listen [::]:443` to `ports.conf` and creating a `<VirtualHost *:443>` or `<VirtualHost [::]:443>` block with `SSLEngine on`, certificate paths, and `SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1`. Enable `mod_ssl` with `a2enmod ssl`. For dual-stack HTTPS, use `*:443` to match both IPv4 and IPv6 Listen directives. Test with `curl -6 https://example.com` and `openssl s_client -connect '[2001:db8::10]:443'`.
