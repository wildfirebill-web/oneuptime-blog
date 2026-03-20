# How to Understand RIPE IPv6 Deployment Requirements

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RIPE, IPv6, ISP, Address Space, RIR, European Network, Deployment

Description: Understand RIPE NCC's IPv6 policies and deployment requirements for ISPs and organizations operating networks in the RIPE NCC service region.

---

RIPE NCC (Réseaux IP Européens Network Coordination Centre) is the Regional Internet Registry (RIR) for Europe, the Middle East, and parts of Central Asia. Understanding RIPE's IPv6 policies helps organizations obtain address space and meet deployment expectations.

## RIPE NCC IPv6 Address Policy

```text
RIPE IPv6 Address Allocation Hierarchy:

IANA
└── RIPE NCC (Regional allocation: 2001:0600::/23, 2001:1400::/23, etc.)
    └── LIR (Local Internet Registry) / ISP: /32 typically
        └── End-user organizations: /48 typically
            └── Site subnet: /64
```

## Obtaining IPv6 Space from RIPE NCC

```text
Requirements to receive IPv6 address space from RIPE NCC:

1. Become an LIR (Local Internet Registry):
   - Annual membership fee to RIPE NCC
   - Minimum: ~€1,400-2,000/year (varies by category)
   - Allows receiving IPv6 (and IPv4) allocations directly

2. Standard IPv6 Allocation:
   - LIRs receive at minimum a /32 IPv6 allocation
   - Can request larger allocations based on documented need
   - Policy: ripe-733 "IPv6 Address Allocation and Assignment Policy"

3. IPv6 PI (Provider Independent) Space:
   - End-sites can receive /48 PI space (ripe-733)
   - Requires assignment for specific site(s)
   - Useful for multi-homed organizations
```

## RIPE Database Registration Requirements

```bash
# After receiving IPv6 allocation, register in RIPE database

# Create inet6num object for your allocation

# Login to: https://apps.db.ripe.net/

# Example inet6num object:
# inet6num: 2001:db8::/32
# netname: EXAMPLE-NET
# country: NL
# org: ORG-EL1-RIPE
# admin-c: ADMIN-RIPE
# tech-c: TECH-RIPE
# mnt-by: EXAMPLE-MNT
# source: RIPE

# Register route6 object for BGP routing
# route6: 2001:db8::/32
# descr: Example IPv6 Route
# origin: AS64500
# mnt-by: EXAMPLE-MNT
# source: RIPE
```

## RIPE IPv6 Deployment Best Practices

```text
RIPE recommends following their "IPv6 Best Current Practices":

1. Address Planning:
   - Use /64 for all end-subnets (never smaller)
   - Use /48 per site
   - /32 per LIR allocation minimum

2. Routing:
   - Advertise your IPv6 prefix consistently
   - Register route6 objects in RIPE DB before advertising
   - Set up IRR (Internet Routing Registry) filters

3. Reverse DNS:
   - Delegate reverse DNS for your IPv6 space
   - Contact RIPE NCC to create ip6.arpa delegation
   - Maintain PTR records for servers

4. Documentation:
   - Keep RIPE DB objects up to date
   - Assign proper tech-c and admin-c contacts
   - Keep mntner and org objects current
```

## RIPE NCC Reverse DNS Delegation

```bash
# Request reverse DNS delegation for IPv6 space
# RIPE NCC auto-delegates for allocations

# Verify delegation exists
dig NS 8.b.d.0.1.0.0.2.ip6.arpa +short

# Set up DNS server for ip6.arpa zone
# Example BIND zone for 2001:db8::/48
# Zone file: /etc/bind/db.8.b.d.0.1.0.0.2.ip6.arpa

# /etc/bind/named.conf.local
zone "8.b.d.0.1.0.0.2.ip6.arpa" {
    type master;
    file "/etc/bind/db.8.b.d.0.1.0.0.2.ip6.arpa";
};
```

## RIPE IPv6 Compliance Requirements for ISPs

```text
ISP membership requirements relevant to IPv6:
- Keep IPv6 allocation in active use
- Register route6 objects for all advertised prefixes
- Maintain accurate RIPE DB objects (inet6num, route6, org)
- Comply with RIPE NCC's accounting and legal requirements

Tools for compliance:
- RIPE NCC Stat: stat.ripe.net (routing analysis)
- RIPE DB query: https://apps.db.ripe.net/db-web-ui/query
- IRRtooler: validate your routing policy
```

## Checking Your RIPE IPv6 Registration

```bash
# Query RIPE database for IPv6 allocation
whois -h whois.ripe.net 2001:db8::/32

# Check route6 registration
whois -h whois.ripe.net -r route6=2001:db8::/32

# Verify routing in looking glasses
# AS-PATH to your prefix: lg.he.net

# Check RPKI status (Route Origin Authorization)
# RIPE NCC provides free RPKI for members
# Create ROA at: https://my.ripe.net/
```

RIPE NCC's IPv6 policies are designed to ensure efficient address use and accurate routing registration, with LIR membership being the typical path for ISPs to obtain IPv6 allocations and the RIPE database serving as the authoritative registry for route6 objects in the RIPE region.
