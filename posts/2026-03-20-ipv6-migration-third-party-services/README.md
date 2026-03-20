# How to Handle IPv6 Migration for Third-Party Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Migration, Third-Party Services, SaaS, Vendor Management

Description: Strategies for handling third-party SaaS services and APIs that lack IPv6 support during IPv6 migration, including vendor engagement, proxying, and NAT64 approaches.

## Introduction

Third-party services — payment gateways, analytics APIs, CDNs, monitoring SaaS, and data providers — are often the last to adopt IPv6. Your migration plan must account for dependencies that cannot be modified. This guide covers how to identify, categorize, and handle third-party IPv6 gaps.

## Step 1: Inventory Third-Party IPv6 Status

```python
#!/usr/bin/env python3
# check_third_party_ipv6.py

import dns.resolver
import socket
import requests
from dataclasses import dataclass

@dataclass
class ThirdPartyStatus:
    name: str
    hostname: str
    has_aaaa: bool
    http_reachable_via_ipv6: bool
    notes: str = ""

THIRD_PARTY_SERVICES = [
    ("Stripe API", "api.stripe.com"),
    ("Twilio API", "api.twilio.com"),
    ("SendGrid", "api.sendgrid.com"),
    ("Datadog", "app.datadoghq.com"),
    ("PagerDuty", "api.pagerduty.com"),
    ("GitHub", "api.github.com"),
    ("AWS S3", "s3.amazonaws.com"),
    ("Slack API", "slack.com"),
    ("Your CRM API", "api.your-crm.com"),
]

results = []
for name, hostname in THIRD_PARTY_SERVICES:
    # Check AAAA record
    has_aaaa = False
    try:
        dns.resolver.resolve(hostname, 'AAAA')
        has_aaaa = True
    except:
        pass

    # Test HTTP reachability over IPv6
    ipv6_reachable = False
    if has_aaaa:
        try:
            resp = requests.get(f"https://{hostname}/", timeout=10,
                                allow_redirects=True)
            ipv6_reachable = True
        except:
            pass

    results.append(ThirdPartyStatus(name, hostname, has_aaaa, ipv6_reachable))

print(f"{'Service':<25} {'AAAA':>6} {'IPv6 HTTP':>10} Status")
print("-" * 55)
for r in results:
    status = "OK" if r.http_reachable_via_ipv6 else ("AAAA-only" if r.has_aaaa else "NO IPv6")
    print(f"{r.name:<25} {'Yes' if r.has_aaaa else 'No':>6} {'Yes' if r.http_reachable_via_ipv6 else 'No':>10}  {status}")
```

## Step 2: Categorize and Plan

| IPv6 Status | Category | Strategy |
|-------------|----------|---------|
| Full IPv6 support | Ready | No action needed |
| AAAA records exist | Partial | Test connectivity, update client config |
| No AAAA, roadmap exists | Planned | Set deadline; escalate if needed |
| No AAAA, no roadmap | Blocked | Use proxy or NAT64 |
| No AAAA, vendor unresponsive | Critical | Find alternative vendor or proxy |

## Strategy A: Direct IPv6 (Best Case)

If a third-party service already has AAAA records, your application can call it over IPv6 automatically via Happy Eyeballs or by forcing IPv6:

```python
import httpx

# Force IPv6 for API call
async with httpx.AsyncClient(transport=httpx.AsyncHTTPTransport(
    local_address="2001:db8::app"  # Bind to IPv6 source
)) as client:
    response = await client.get("https://api.example-with-ipv6.com/v1/data")
```

## Strategy B: Outbound Proxy for IPv4-Only APIs

When your application (IPv6-addressed) needs to call IPv4-only external APIs, add an outbound IPv4 proxy:

```nginx
# Squid or Nginx proxy for outbound IPv4 calls
# nginx as forward proxy (Nginx Plus or 3rd-party module)

# Alternative: use HAProxy as a forward proxy
# /etc/haproxy/haproxy-outbound.cfg

frontend outbound_proxy
    bind [::]:3128          # Accept IPv6 connections from app
    mode tcp
    default_backend ipv4_egress

backend ipv4_egress
    mode tcp
    server egress1 203.0.113.1:3128  # IPv4 egress proxy
```

## Strategy C: NAT64 for Systematic IPv4 Egress

```bash
# If running IPv6-only internally and needing IPv4 egress for all services:

# Option 1: Jool NAT64 (kernel module)
apt-get install jool-dkms jool-tools

# Configure Jool NAT64
jool instance add "default" --netfilter --pool6 64:ff9b::/96

# Add route for NAT64 prefix
ip -6 route add 64:ff9b::/96 dev lo

# Configure DNS64
# Named or unbound: synthesize AAAA for IPv4-only domains
# All IPv4-only external services resolve as [64:ff9b::a.b.c.d]
# NAT64 translates to 203.0.113.5 (real IPv4 address)
```

## Strategy D: Vendor Engagement Template

```markdown
# Subject: IPv6 Support Request for [Service Name]

Dear [Vendor] Technical Team,

We are migrating our infrastructure to IPv6 dual-stack by Q3 2026 and
require IPv6 support from our critical vendors.

**Request:**
1. Does [Service Name] currently support IPv6? (AAAA records, IPv6 API endpoints)
2. If not, what is your IPv6 roadmap timeline?
3. Is there an interim IPv6 API endpoint we can test against?

**Impact:**
Our applications will transition to IPv6-only networking in Q4 2026.
Services without IPv6 support will require us to implement workarounds
(proxies, NAT64) which add latency and operational complexity.

We are evaluating alternatives if IPv6 support is not available by Q4 2026.

Please respond by [DATE] to help us plan accordingly.
```

## Tracking Third-Party IPv6 Gaps

```markdown
# Third-Party IPv6 Gap Tracker

| Service | Status | Workaround | Resolution | Owner |
|---------|--------|------------|------------|-------|
| Payment gateway | Blocked | Outbound proxy | Q3 2026 | Platform team |
| Analytics API | Partial | None needed | Ready | N/A |
| Email API | Blocked | NAT64 | Vendor Q4 2026 | Platform team |
| CDN | Ready | N/A | Done | N/A |
```

## Conclusion

Third-party services without IPv6 support require proxying or NAT64 — both are viable but add operational complexity. Inventory all third-party dependencies early (Phase 1 of migration) and engage vendors with IPv6 gaps immediately. The vendor engagement template provides a professional basis for escalation. For services without a credible IPv6 roadmap, NAT64 provides transparent translation for outbound calls and is preferable to indefinitely maintaining a patchwork of individual outbound proxies.
