# How to Configure Cloudflare IPv6 Compatibility

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cloudflare, IPv6, CDN, DNS, Compatibility, Web

Description: A guide to Cloudflare's IPv6 compatibility feature, including how to enable it, how it works, and how it provides IPv6 access to IPv4-only origins.

Cloudflare offers two IPv6 settings: **IPv6 Compatibility** (automatic IPv6 for all Cloudflare-proxied records) and manual IPv6 record management. When IPv6 compatibility is enabled, Cloudflare provides a AAAA record for your domain even if your origin server only has an IPv4 address.

## How Cloudflare IPv6 Compatibility Works

```
IPv6 Client → Cloudflare Edge (IPv6) → Cloudflare Network → Your Origin (IPv4)
              ↑ Cloudflare assigns IPv6 address to your domain
              ↑ Translates IPv6 client connection to IPv4 origin connection
```

Your origin server remains IPv4-only. Cloudflare handles the translation at its edge.

## Enabling IPv6 Compatibility

### Via Cloudflare Dashboard

1. Log in to Cloudflare Dashboard
2. Select your domain
3. Navigate to **Network** tab
4. Toggle **IPv6 Compatibility** to **On**

### Via Cloudflare API

```bash
# Enable IPv6 Compatibility via API
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/{zone_id}/settings/ipv6" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "on"}'

# Verify the setting
curl "https://api.cloudflare.com/client/v4/zones/{zone_id}/settings/ipv6" \
  -H "Authorization: Bearer $CF_API_TOKEN"
```

### Via Terraform (Cloudflare Provider)

```hcl
resource "cloudflare_zone_settings_override" "example" {
  zone_id = var.zone_id

  settings {
    ipv6 = "on"
  }
}
```

## Verifying IPv6 Compatibility

```bash
# After enabling, Cloudflare should provide AAAA records
dig AAAA example.com

# Expected: AAAA record pointing to Cloudflare's IPv6 range
# example.com. 300 IN AAAA 2606:4700:3033::xxxx

# Test IPv6 connection
curl -6 https://example.com/

# Check that origin receives connection (via Cloudflare dashboard or logs)
```

## IPv6 with Cloudflare DNS-Only Records (Orange Cloud Off)

When the orange cloud is off (DNS only, no proxy), you manage IPv6 directly:

```hcl
# Direct AAAA record (bypasses Cloudflare proxy)
resource "cloudflare_record" "aaaa_direct" {
  zone_id = var.zone_id
  name    = "direct"
  type    = "AAAA"
  value   = "2001:db8::your-server"
  proxied = false    # DNS only, no Cloudflare proxy
  ttl     = 300
}

# Proxied AAAA record (through Cloudflare)
resource "cloudflare_record" "aaaa_proxied" {
  zone_id = var.zone_id
  name    = "@"
  type    = "AAAA"
  value   = "2001:db8::your-server"
  proxied = true     # Traffic goes through Cloudflare
  ttl     = 1        # Auto TTL when proxied
}
```

## Cloudflare Page Rules and IPv6

Cloudflare page rules apply equally to IPv4 and IPv6 client connections:

```bash
# Create a page rule that redirects IPv6 clients (same as IPv4)
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/pagerules" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "targets": [{"target": "url", "constraint": {"operator": "matches", "value": "http://example.com/*"}}],
    "actions": [{"id": "always_use_https"}],
    "status": "active"
  }'
```

## Checking Client IPv6 in Cloudflare Analytics

Cloudflare Analytics shows the distribution of IPv4 vs IPv6 traffic:

1. Cloudflare Dashboard → Analytics & Logs → Traffic
2. Look for "Requests by IP version"
3. Compare IPv4 and IPv6 request percentages

## IPv6 Client IP in Cloudflare Workers

```javascript
// Access the client's IPv6 address in a Cloudflare Worker
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const clientIP = request.headers.get('CF-Connecting-IP');
  const ipVersion = clientIP.includes(':') ? 'IPv6' : 'IPv4';

  return new Response(`Your IP: ${clientIP} (${ipVersion})`);
}
```

Cloudflare's IPv6 Compatibility feature is the zero-effort path to IPv6 support — a single toggle in the dashboard provides IPv6 access to your domain without any changes to your origin infrastructure.
