# How to Configure Akamai CDN for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Akamai, IPv6, CDN, Edge, Enterprise, Content Delivery

Description: A guide to configuring Akamai CDN for IPv6 delivery, including edge server IPv6 support, origin connectivity, and IPv6 client handling in Akamai properties.

Akamai has supported IPv6 since 2011 and has one of the largest dual-stack edge networks globally. Configuring IPv6 in Akamai involves enabling dual-stack delivery in property configurations and ensuring origin servers accept Akamai's IPv6 connections.

## Akamai IPv6 Architecture

```
IPv6 Client → Akamai Edge (dual-stack) → Origin (IPv4 or IPv6)
              ↑ Akamai has both A and AAAA in its edge CNAMEs
              ↑ Client uses IPv6 when available (Happy Eyeballs)
```

## Enabling IPv6 in Akamai Property Manager

Configuration is done in Akamai Control Center under Property Manager:

### Via Akamai APIs (PAPI)

```json
{
  "name": "IPv6 Delivery",
  "options": {
    "enabled": true,
    "ipVersion": "IPV6",
    "enableIpv6": true,
    "dualStack": true
  }
}
```

### Property Manager Behavior

In the Property Manager UI:
1. Navigate to your property
2. Add behavior: **IP/Geo ACL** or use the **Origin server** behavior
3. In **Origin Server** settings, set IP version preference to "IPv4+IPv6 (Dual Stack)"

## Akamai Origin Connectivity

Configure how Akamai connects to your origin:

```json
{
  "name": "Origin Connectivity",
  "behaviors": [
    {
      "name": "origin",
      "options": {
        "originType": "CUSTOMER",
        "hostname": "origin.example.com",
        "ipVersion": "IPV4_IPV6",
        "enableTrueClientIp": true,
        "trueClientIpHeader": "True-Client-IP",
        "trueClientIpClientSetting": true
      }
    }
  ]
}
```

## Akamai CLI Configuration

```bash
# Install Akamai CLI
brew install akamai/tap/akamai   # macOS
# or: pip install akamai-edgegrid

# Authenticate
akamai auth

# List properties
akamai property list-property-names

# Get property rules (includes IPv6 settings)
akamai property get-property-rules \
  --property-name example.com \
  --output-file rules.json

# Modify IPv6 settings and update property
# Edit rules.json to add IPv6 behavior
akamai property update-property-rules \
  --property-name example.com \
  --input-file rules.json
```

## IPv6 Client IP in Akamai

Akamai forwards the real client IPv6 in multiple headers:

```bash
# Headers your origin receives:
# True-Client-IP: 2001:db8::client       (Akamai header, if configured)
# X-Forwarded-For: 2001:db8::client, akamai.edge.ip
# X-Real-IP: 2001:db8::client

# Configure nginx to trust Akamai IP ranges
# Get current Akamai edge IP ranges from:
# https://techdocs.akamai.com/origin-ip-acl/docs/akamai-ip-ranges
```

## Testing Akamai IPv6 Delivery

```bash
# Verify AAAA records for Akamai-hosted domain
dig AAAA example.com

# Test IPv6 delivery
curl -6 -v https://example.com/ 2>&1 | head -20

# Check Akamai edge response headers
curl -6 -D - https://example.com/ -o /dev/null | grep -i "x-akamai\|x-check\|server"

# Edge server information headers:
# X-Cache: TCP_HIT from a2-xxx-xxx-xxx.deploy.akamaitechnologies.com
# X-Check-Cacheable: YES
```

## Akamai Site Shield (IPv6 Origin Protection)

Site Shield restricts origin access to Akamai's IPs only:

```bash
# Get Akamai Site Shield IPv6 ranges
curl https://api.akamai.com/siteshield/v1/maps \
  -H "Authorization: ..." | python3 -m json.tool | grep ipv6

# Configure your firewall to allow only these IPv6 ranges
sudo ip6tables -A INPUT -p tcp --dport 443 \
  -s 2a02:26f0::/32 -j ACCEPT   # Example Akamai IPv6 range
```

## Monitoring IPv6 Delivery in Akamai

```bash
# Using Akamai Reporting API
curl "https://developer.akamai.com/api/data_services/reporting/v1/delivery-quality/data" \
  -H "Authorization: ..." \
  --data '{
    "metrics": ["ipv6RequestsTotal", "ipv4RequestsTotal"],
    "dimensions": ["time"],
    "filters": {"property": ["example.com"]}
  }'
```

Akamai's extensive IPv6 edge network and built-in dual-stack delivery support means that enabling IPv6 primarily involves a configuration change in Property Manager rather than infrastructure changes, with the edge network handling the complexity.
