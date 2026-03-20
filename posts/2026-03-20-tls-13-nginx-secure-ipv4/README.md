# How to Configure TLS 1.3 on Nginx for Secure IPv4 Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TLS, Nginx, SSL, HTTPS, Security, IPv4

Description: Learn how to configure TLS 1.3 on Nginx to enforce modern cipher suites, disable older TLS versions, and serve HTTPS traffic securely on IPv4.

## Why TLS 1.3?

TLS 1.3 offers significant improvements over TLS 1.2:
- **Faster handshake:** 1-RTT (vs 2-RTT for TLS 1.2) with optional 0-RTT resumption
- **Stronger security:** Only strong cipher suites supported, no negotiation needed
- **Forward secrecy:** All key exchanges use ephemeral keys
- **Simpler:** Removes many legacy options that created vulnerabilities

Nginx 1.13.0+ with OpenSSL 1.1.1+ supports TLS 1.3.

## Step 1: Verify Nginx and OpenSSL Support

```bash
# Check Nginx version and TLS support
nginx -V 2>&1 | grep -E "version|TLS|ssl"

# Check OpenSSL version (must be 1.1.1+)
openssl version

# Verify TLS 1.3 ciphersuites are available
openssl ciphers -v TLSv1.3
```

## Step 2: Configure TLS 1.3 in Nginx

Edit your Nginx SSL server block configuration:

```nginx
# /etc/nginx/conf.d/secure-site.conf

server {
    listen 443 ssl;
    server_name example.com www.example.com;

    # Certificate and private key
    ssl_certificate     /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # Enable TLS 1.3 only (remove TLSv1.2 if you want TLS 1.3 only)
    # To support older clients, include both:
    ssl_protocols TLSv1.2 TLSv1.3;

    # TLS 1.3 cipher suites (set automatically by OpenSSL for TLS 1.3)
    # For TLS 1.2 fallback, specify strong cipher suites:
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;

    # Prefer server cipher order
    ssl_prefer_server_ciphers off;   # For TLS 1.3, disable server preference

    # Enable session caching for performance
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;         # Disable for perfect forward secrecy

    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/certs/chain.crt;

    # DNS resolver for OCSP
    resolver 1.1.1.1 8.8.8.8 valid=300s;
    resolver_timeout 5s;

    # Security headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;

    root /var/www/html;
    index index.html;
}

# HTTP to HTTPS redirect
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}
```

## Step 3: Generate a Strong DH Parameters File

For TLS 1.2 fallback, use 2048-bit DH parameters:

```bash
# Generate DH parameters (takes a few minutes)
openssl dhparam -out /etc/nginx/dhparam.pem 2048
```

Add to nginx config:

```nginx
ssl_dhparam /etc/nginx/dhparam.pem;
```

## Step 4: Test and Reload Nginx

```bash
# Test configuration syntax
sudo nginx -t

# Reload without dropping connections
sudo systemctl reload nginx
```

## Step 5: Verify TLS 1.3 Is Active

```bash
# Test TLS 1.3 handshake
openssl s_client -connect example.com:443 -tls1_3 2>&1 | grep -E "Protocol|Cipher"

# Expected output:
# Protocol  : TLSv1.3
# Cipher    : TLS_AES_256_GCM_SHA384

# Test with curl
curl -vI --tlsv1.3 https://example.com 2>&1 | grep -E "TLS|SSL"
```

## Step 6: Check SSL Rating

Run your site through SSL Labs to verify the configuration:

```bash
# Using sslyze for local testing
pip install sslyze
python3 -m sslyze --regular example.com:443

# Or use testssl.sh
./testssl.sh example.com
```

An A+ rating requires TLS 1.3, strong ciphers, HSTS, and no known vulnerabilities.

## Conclusion

Configuring TLS 1.3 on Nginx requires Nginx 1.13+ with OpenSSL 1.1.1+. Set `ssl_protocols TLSv1.2 TLSv1.3` for broad compatibility, disable TLS 1.0/1.1, configure strong TLS 1.2 ciphers for fallback, enable OCSP stapling, and add HSTS headers. Verify with `openssl s_client -tls1_3` and test your SSL rating with sslyze or testssl.sh.
