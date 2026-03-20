# How to Set Up HTTP/2 on Nginx with TLS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, HTTP/2, TLS, HTTPS, Performance, Web Server

Description: Learn how to enable HTTP/2 on Nginx with TLS to take advantage of multiplexing and header compression for faster web applications.

## Why HTTP/2?

HTTP/2 introduces multiplexing (multiple requests over a single connection), header compression (HPACK), and server push-dramatically reducing page load times compared to HTTP/1.1. Nginx supports HTTP/2 but requires TLS in practice because all major browsers only implement HTTP/2 over HTTPS.

## Prerequisites

- Nginx 1.9.5 or later (compiled with `ngx_http_v2_module`)
- A valid TLS certificate (Let's Encrypt works well)

Verify your Nginx build includes HTTP/2 support:

```bash
nginx -V 2>&1 | grep http_v2
```

You should see `--with-http_v2_module` in the output.

## Enabling HTTP/2

Add `http2` to the `listen` directive on port 443:

```nginx
# /etc/nginx/sites-available/example.com

server {
    # Enable HTTP/2 by adding the http2 parameter
    listen 443 ssl;
    http2 on;
    listen [::]:443 ssl;

    server_name example.com www.example.com;

    # TLS certificate paths (Let's Encrypt example)
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/html;
    index index.html;
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

Note: In Nginx 1.25.1+, `http2 on` is a standalone directive. In older versions, use `listen 443 ssl http2;`.

## Recommended TLS Configuration for HTTP/2

HTTP/2 mandates TLS 1.2 or higher and a strong cipher suite. Apply these settings:

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Require TLS 1.2 or 1.3 (HTTP/2 mandates this)
    ssl_protocols TLSv1.2 TLSv1.3;

    # HTTP/2-compatible cipher suites
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # Enable OCSP stapling for faster TLS handshakes
    ssl_stapling on;
    ssl_stapling_verify on;

    # Session resumption reduces handshake overhead
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    add_header Strict-Transport-Security "max-age=63072000" always;

    root /var/www/html;
}
```

## Verifying HTTP/2 Is Active

After reloading Nginx (`nginx -s reload`), verify with `curl`:

```bash
# --http2 requests HTTP/2; -I fetches headers only
curl -I --http2 https://example.com
```

Look for `HTTP/2 200` in the output. Alternatively, use the browser DevTools Network tab-the Protocol column should show `h2`.

## Tuning HTTP/2 Parameters

```nginx
http {
    # Number of concurrent streams per connection (default: 128)
    http2_max_concurrent_streams 256;

    # Chunk size for HTTP/2 data frames (default: 16k)
    http2_chunk_size 16k;
}
```

## Conclusion

Enabling HTTP/2 on Nginx is as simple as adding `http2 on` and ensuring a solid TLS configuration. Combined with OCSP stapling and session resumption, you'll see measurable improvements in Time to First Byte and overall page load performance.
