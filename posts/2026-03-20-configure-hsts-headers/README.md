# How to Configure HSTS (HTTP Strict Transport Security) Headers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: HSTS, HTTP, HTTPS, Security, Nginx, Apache, Header

Description: Learn how to implement HTTP Strict Transport Security headers to enforce HTTPS connections at the browser level, protecting against SSL stripping attacks.

## What Is HSTS?

HSTS (HTTP Strict Transport Security) is a response header that instructs browsers to only connect to your site via HTTPS for a specified duration. Once a browser receives an HSTS header:
- All future connections to your domain are automatically upgraded to HTTPS
- The browser refuses to load the site over plain HTTP (shows an error instead of silently redirecting)
- This protects against SSL stripping attacks where a man-in-the-middle downgrades HTTPS to HTTP

## HSTS Header Format

```text
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

| Parameter | Description |
|---|---|
| `max-age=N` | Browser remembers HTTPS-only for N seconds |
| `includeSubDomains` | Apply to all subdomains |
| `preload` | Opt into browser preload lists |

## Step 1: Add HSTS to Nginx

```nginx
# /etc/nginx/conf.d/example.com.conf

server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # HSTS - 1 year, include subdomains, eligible for preloading
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # The 'always' keyword ensures the header is added even for error responses
}
```

## Step 2: Add HSTS to Apache

```apache
<VirtualHost *:443>
    ServerName example.com

    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

    # Ensure mod_headers is enabled
    # sudo a2enmod headers
</VirtualHost>
```

## Step 3: Progressive HSTS Deployment

HSTS can be difficult to remove once deployed with a long max-age. Deploy progressively:

```bash
# Week 1: Short max-age (5 minutes) - test for issues

add_header Strict-Transport-Security "max-age=300" always;

# Week 2: Increase to 1 day - still reversible
add_header Strict-Transport-Security "max-age=86400" always;

# Week 3: 1 week
add_header Strict-Transport-Security "max-age=604800" always;

# Week 4: Add includeSubDomains (make sure ALL subdomains have HTTPS)
add_header Strict-Transport-Security "max-age=604800; includeSubDomains" always;

# Month 2: Production value - 1 year + preload eligible
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

## Step 4: Verify HSTS Header Is Set

```bash
# Check that HSTS header appears in the response
curl -sI https://example.com/ | grep -i strict

# Expected output:
# strict-transport-security: max-age=31536000; includeSubDomains; preload

# Verify it's also on subdomains if using includeSubDomains
curl -sI https://api.example.com/ | grep -i strict

# Test with verbose output
curl -v https://example.com/ 2>&1 | grep -A2 "strict-transport"
```

## Step 5: Submit to HSTS Preload List

The HSTS preload list is a browser-maintained list of domains that are hardcoded to always use HTTPS, even on the very first visit. Requirements:
1. Valid HTTPS certificate
2. HTTPS redirect from HTTP
3. HSTS header with `max-age ≥ 31536000`, `includeSubDomains`, and `preload`
4. All subdomains must be HTTPS-capable

Submit at: https://hstspreload.org/

```bash
# Verify eligibility before submitting
curl -sI https://example.com/ | grep strict-transport-security

# hstspreload.org also provides a JSON API to check status
curl "https://hstspreload.org/api/v2/status?domain=example.com"
```

## Step 6: Handle HSTS with Internal/Dev Environments

Be careful with `includeSubDomains` if you have internal subdomains:

```nginx
# Production - full HSTS with subdomains and preload
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

# Staging - shorter max-age without includeSubDomains
# (so staging subdomain issues don't affect the main domain)
add_header Strict-Transport-Security "max-age=86400" always;

# Development - no HSTS (allows HTTP locally)
# Don't add the header in development environments
```

## Revoking HSTS (Emergency Procedure)

If you need to remove HSTS:

```bash
# Set max-age to 0 to tell browsers to forget HSTS immediately
add_header Strict-Transport-Security "max-age=0" always;

# Note: Browsers that visited before may still enforce HSTS until the original
# max-age expires. For preloaded domains, removal from preload list takes months.
```

## Conclusion

HSTS prevents SSL stripping attacks by instructing browsers to only connect via HTTPS. Deploy progressively starting with short max-age values, add `includeSubDomains` only when all subdomains have valid HTTPS, and submit to the preload list for maximum protection. Use `max-age=0` to revoke HSTS if you need to go back to HTTP (though preloaded domains cannot be quickly removed).
