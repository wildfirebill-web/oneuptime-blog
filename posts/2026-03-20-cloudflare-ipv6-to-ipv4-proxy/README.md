# How to Configure Cloudflare IPv6-to-IPv4 Origin Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cloudflare, IPv6, IPv4, Proxy, CDN, Origin

Description: A guide to using Cloudflare as an IPv6-to-IPv4 proxy, allowing IPv6 clients to reach IPv4-only origin servers through Cloudflare's edge network.

Cloudflare's proxy mode automatically acts as an IPv6-to-IPv4 translator when your origin server only has an IPv4 address. IPv6 clients connect to Cloudflare's edge over IPv6, and Cloudflare connects to your origin over IPv4 - providing seamless IPv6 access to IPv4-only infrastructure.

## IPv6-to-IPv4 Proxy Architecture

```text
IPv6 Client                          Your IPv4 Origin
    |                                      |
    | HTTPS over IPv6                      |
    ↓                                      |
[Cloudflare Edge]  ←─── IPv4 ────────────→|
    | • Accepts IPv6 from clients          |
    | • Connects to origin via IPv4        |
    | • Full DDoS protection applies       |
    | • Caching at edge                    |
```

## Prerequisites

For the IPv6-to-IPv4 proxy to work:
- Your DNS records must be **proxied** (orange cloud in Cloudflare)
- IPv6 Compatibility must be enabled in Network settings
- Your origin only needs an IPv4 address

## Enabling the Proxy

```bash
# Step 1: Ensure your A record is proxied (orange cloud)

curl -X PATCH "https://api.cloudflare.com/client/v4/zones/{zone_id}/dns_records/{record_id}" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"proxied": true}'

# Step 2: Enable IPv6 Compatibility
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/{zone_id}/settings/ipv6" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "on"}'
```

### Terraform Configuration

```hcl
# Only an A record needed - Cloudflare provides AAAA automatically
resource "cloudflare_record" "main" {
  zone_id = var.zone_id
  name    = "@"
  type    = "A"
  value   = "203.0.113.10"    # Your IPv4-only origin
  proxied = true               # Enable Cloudflare proxy (orange cloud)
}

# Enable IPv6 compatibility
resource "cloudflare_zone_settings_override" "main" {
  zone_id = var.zone_id
  settings {
    ipv6 = "on"
  }
}
```

## What Cloudflare Does Automatically

When the proxy is enabled with IPv6 compatibility:

1. Cloudflare adds `2606:4700::/32` AAAA records for your domain
2. IPv6 clients resolve the AAAA record and connect to Cloudflare's IPv6 address
3. Cloudflare forwards the request to your IPv4 origin
4. Cloudflare adds `CF-Connecting-IP` header with the real IPv6 client address

## Reading IPv6 Client IPs at Your Origin

Your origin receives connections from Cloudflare's IPv4 addresses, but the real client's IPv6 is in the `CF-Connecting-IP` header:

```nginx
# /etc/nginx/nginx.conf

# Trust Cloudflare IP ranges
# IPv4 ranges
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
# ... (full list at https://www.cloudflare.com/ips-v4)

# IPv6 ranges (when Cloudflare connects from IPv6)
set_real_ip_from 2400:cb00::/32;
set_real_ip_from 2606:4700::/32;

# Use CF-Connecting-IP for real client IP (includes IPv6)
real_ip_header CF-Connecting-IP;

# Now $remote_addr shows the real client IPv6 address
log_format cloudflare '$remote_addr - $request - $status';
```

## Restricting Direct IPv4 Access to Origin

Since Cloudflare proxies IPv6 clients to your IPv4 origin, consider allowing only Cloudflare IPs at your origin firewall:

```bash
# Allow only Cloudflare's IPv4 ranges to reach origin
# Get current Cloudflare IPs:
curl https://www.cloudflare.com/ips-v4

# Example ip6tables rule (block direct IPv6 access - all IPv6 goes through Cloudflare)
sudo ip6tables -A INPUT -p tcp --dport 443 ! -s 2606:4700::/32 -j DROP
```

## Testing IPv6-to-IPv4 Proxy

```bash
# Verify AAAA record is Cloudflare's
dig AAAA example.com
# Should return 2606:4700:xxxx::xxxx

# Test IPv6 client connection
curl -6 -v https://example.com/ 2>&1 | grep "Connected to"
# Should connect to a Cloudflare IPv6 address

# Verify CF-Connecting-IP at origin
# Check your origin access logs for CF-Connecting-IP values
# Should contain the real client IPv6 address
```

Cloudflare's IPv6-to-IPv4 proxy is the standard approach for legacy IPv4 infrastructure - one DNS change and one setting toggle gives full IPv6 connectivity to clients worldwide without any origin changes.
