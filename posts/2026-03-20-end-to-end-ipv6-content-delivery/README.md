# How to Set Up End-to-End IPv6 Content Delivery (DNS + CDN + Origin)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, CDN, DNS, Origin, End-to-End, Content Delivery

Description: A guide to setting up a complete end-to-end IPv6 content delivery pipeline from DNS resolution through CDN edge to origin server, ensuring IPv6 connectivity at every layer.

End-to-end IPv6 content delivery means every hop in the request chain supports IPv6: the DNS resolver returns AAAA records, the CDN edge accepts IPv6 connections, and the CDN connects to the origin over IPv6. This guide walks through the complete setup.

## Architecture Overview

```
Client (IPv6)
    ↓ DNS AAAA query
Recursive Resolver (IPv6 capable)
    ↓ Returns 2001:db8::cdn AAAA
CDN Edge (IPv6 listener)
    ↓ Connect to origin via IPv6
Origin Server (IPv6 enabled)
```

## Step 1: Origin Server IPv6 Setup

```bash
# Ensure origin has an IPv6 address
ip -6 addr show eth0
# Must show a global unicast address (2001: or similar)

# Configure nginx to listen on IPv6
# /etc/nginx/sites-enabled/app
server {
    listen 80;
    listen [::]:80;           # IPv6 listener
    listen 443 ssl;
    listen [::]:443 ssl;      # IPv6 HTTPS listener

    server_name origin.example.com;

    location / {
        root /var/www/html;
        index index.html;
    }
}

# Test IPv6 listener
ss -6 -tlnp | grep nginx
```

## Step 2: Origin DNS (AAAA Record)

```bash
# Add AAAA record for origin
dig AAAA origin.example.com   # Should return the origin's IPv6 address

# Verify from CDN's perspective
# CDN must be able to resolve AAAA for the origin hostname
# Test from a CDN probe server:
nslookup -type=AAAA origin.example.com
```

## Step 3: CDN Configuration with IPv6 Origin

### Cloudflare

```bash
# Cloudflare connects to origin via IPv6 when AAAA exists
# Set origin in Cloudflare Dashboard:
# Name: origin.example.com (has both A and AAAA record)
# Cloudflare auto-selects IPv6 if available

# Verify in Cloudflare: Network tab → show 'Resolved via IPv6'
```

### AWS CloudFront

```hcl
resource "aws_cloudfront_distribution" "ipv6_e2e" {
  enabled         = true
  is_ipv6_enabled = true    # Accept IPv6 from clients

  origin {
    # Hostname with AAAA record — CloudFront connects via IPv6
    domain_name = "origin.example.com"
    origin_id   = "IPv6Origin"

    custom_origin_config {
      http_port  = 80
      https_port = 443
      origin_protocol_policy = "https-only"
    }
  }
  ...
}
```

### Fastly

```hcl
resource "fastly_service_vcl" "ipv6_e2e" {
  name = "ipv6-e2e"

  domain { name = "cdn.example.com" }

  backend {
    # Origin with IPv6 — Fastly connects via IPv6 automatically
    address = "origin.example.com"
    name    = "ipv6_origin"
    port    = 443
    use_ssl = true
  }
  ...
}
```

## Step 4: CDN Edge IPv6 DNS Records

```bash
# CDN automatically provides AAAA for your CDN hostname
# After pointing cdn.example.com to your CDN:

dig AAAA cdn.example.com
# Cloudflare: 2606:4700::xxxx
# CloudFront: 2600:9000::xxxx
# Fastly: 2a04:4e40::xxxx
```

## Step 5: Your Domain's DNS Records

```hcl
# DNS records: both A and AAAA pointing to CDN
resource "cloudflare_record" "a" {
  zone_id = var.zone_id
  name    = "@"
  type    = "A"
  value   = "203.0.113.cdn"  # CDN IPv4
  proxied = true
}

resource "cloudflare_record" "aaaa" {
  zone_id = var.zone_id
  name    = "@"
  type    = "AAAA"
  value   = "2001:db8::cdn"  # CDN IPv6
  proxied = true
}

# Or use CNAME to CDN (CDN provides both A and AAAA)
resource "cloudflare_record" "cname" {
  zone_id = var.zone_id
  name    = "cdn"
  type    = "CNAME"
  value   = "xxxx.cloudfront.net"
  proxied = false
}
```

## Verification: Test End-to-End IPv6

```bash
# Complete end-to-end verification

# 1. Verify DNS returns AAAA
dig AAAA example.com
# Must return an IPv6 address

# 2. Test IPv6 client connection
curl -6 -v https://example.com/ 2>&1 | grep "Connected to"
# Must show IPv6 address

# 3. Verify CDN is in the path (check headers)
curl -6 https://example.com/ -D -
# Cloudflare: CF-RAY header
# CloudFront: X-Amz-Cf-Id header
# Fastly: X-Served-By header

# 4. Verify CDN connects to origin via IPv6
# Check origin access logs — CDN should connect from IPv6 address
tail -f /var/log/nginx/access.log | grep "::"

# 5. Performance: measure IPv6 connection latency
curl -6 -w "DNS: %{time_namelookup}s, Connect: %{time_connect}s, TTFB: %{time_starttransfer}s\n" \
  -o /dev/null https://example.com/
```

## Troubleshooting E2E IPv6

| Layer | Check | Command |
|---|---|---|
| Origin IPv6 | Has IPv6 address? | `ip -6 addr show` |
| Origin DNS | Has AAAA record? | `dig AAAA origin.example.com` |
| CDN config | IPv6 origin enabled? | CDN dashboard |
| CDN DNS | Has AAAA record? | `dig AAAA cdn.example.com` |
| Client | Gets AAAA? | `dig AAAA example.com` |
| E2E | Connection via IPv6? | `curl -6 -v https://example.com` |

End-to-end IPv6 content delivery requires IPv6 at every layer — origin server, CDN edge, and DNS — but once properly configured, provides optimal latency for IPv6 clients by leveraging native IPv6 routing across the entire path.
