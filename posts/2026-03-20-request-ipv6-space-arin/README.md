# How to Request IPv6 Address Space from ARIN

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ARIN, RIR, Address Space, IP Allocation

Description: Request IPv6 address space from ARIN (American Registry for Internet Numbers) including eligibility requirements, application process, and required documentation.

## ARIN IPv6 Allocation Overview

ARIN serves the United States, Canada, and many Caribbean and North Atlantic territories. IPv6 allocations from ARIN come in two forms:

| Type | Who Gets It | Typical Size | Requirement |
|---|---|---|---|
| Direct allocation | ISPs, large orgs | /32 | ARIN member, plan to assign to customers |
| Direct assignment | End users | /48 | Justify need for /48 or larger |
| ISP sub-allocation | ISP customers | /48, /56, /64 | From ISP's ARIN allocation |

## Eligibility Requirements

```
For a /32 Direct Allocation (ISPs):
  - Be an ARIN member (annual fee)
  - Have an IPv4 allocation from ARIN, or
  - Plan to assign IPv6 to at least 200 customers within 5 years
  - Demonstrate network infrastructure

For a /48 Direct Assignment (End Users):
  - Have existing IPv4 space from ARIN, or
  - Need multiple /64 subnets
  - Demonstrate technical justification

Fees (2024):
  X-Small: $500/year (orgs with /32)
  Small:   $1,000/year (ISPs with /28)
  Medium:  $2,000/year (ISPs with /24)
```

## Application Process

```
Step 1: Create ARIN Online account
  - Go to https://account.arin.net
  - Register organization

Step 2: Become an ARIN member (if ISP)
  - Submit membership application
  - Pay annual fee

Step 3: Submit IPv6 request
  - Login to ARIN Online
  - Navigate: Requests → IPv6 → Direct Allocation
  - Fill in:
    * Organization information
    * Network infrastructure details
    * Customer projection (for ISPs)
    * Intended use

Step 4: ARIN staff review
  - Review typically takes 1-5 business days
  - May request additional documentation

Step 5: Receive allocation
  - ARIN assigns prefix from 2600::/12 (ARIN's block)
  - Update ARIN Whois database
  - Create ROA in RPKI
```

## RPKI: Create Route Origin Authorization

```bash
# After receiving ARIN allocation, create RPKI ROA
# Login to ARIN Online → RPKI → Create ROA

# Or use ARIN's API
# (requires ARIN API key)

# ROA fields:
# ASN:       65001 (your AS number)
# Prefix:    2600:db8::/32 (your ARIN allocation)
# Max length: 48 (allow /48 announcements)

# Verify ROA in RPKI
rpki-validator-cli --prefix 2600:db8::/32 --asn 65001

# Check RPKI validity
curl -s "https://api.bgpstuff.net/rpki?prefix=2600:db8::/32&asn=65001"
```

## ARIN Whois Registration

```
# After allocation, update ARIN Whois (ARIN Online)

# Network object example:
# inetnum: 2600:db8::/32
# netname: COMPANY-IPV6
# descr:   Example Company IPv6 Space
# country: US
# admin-c: ADMIN-ARIN
# tech-c:  TECH-ARIN
# status:  ALLOCATED PA
# source:  ARIN

# DNS Reverse Delegation:
# ARIN delegates 8.b.d.0.0.0.6.2.ip6.arpa to your nameservers
# Submit NS records via ARIN Online → Reverse DNS
```

## Post-Allocation Checklist

```bash
#!/bin/bash
# post-arin-checklist.sh — Validate new ARIN allocation

YOUR_PREFIX="2600:db8::/32"
YOUR_ASN="65001"

echo "=== Post-ARIN Allocation Checklist ==="

# 1. Verify prefix in ARIN Whois
echo "1. Checking ARIN Whois..."
whois -h whois.arin.net "${YOUR_PREFIX}" | grep "NetName\|CIDR"

# 2. Verify BGP announcement
echo "2. Checking BGP announcement..."
curl -s "https://api.bgpstuff.net/announced?prefix=${YOUR_PREFIX}" | python3 -m json.tool

# 3. Verify RPKI ROA
echo "3. Checking RPKI..."
curl -s "https://api.bgpstuff.net/rpki?prefix=${YOUR_PREFIX}&asn=${YOUR_ASN}"

# 4. Verify reverse DNS delegation
echo "4. Checking reverse DNS..."
dig NS 8.b.d.0.0.0.6.2.ip6.arpa +short

echo ""
echo "Complete: ARIN allocation checklist done"
```

## Conclusion

Requesting IPv6 from ARIN requires ARIN membership for ISPs and a technical justification for the requested prefix size. ISPs typically receive /32 allocations (which they sub-allocate as /48s to customers), while end-user organizations receive /48 direct assignments. After approval, complete three critical post-allocation steps: create an RPKI Route Origin Authorization (ROA) to prevent route hijacking, update ARIN Whois with accurate network registration data, and request reverse DNS delegation for your ip6.arpa zone. ARIN's review process typically takes 1-5 business days for straightforward requests.
