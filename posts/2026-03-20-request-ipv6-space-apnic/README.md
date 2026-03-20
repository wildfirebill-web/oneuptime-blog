# How to Request IPv6 Address Space from APNIC - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, APNIC, RIR, Address Space, IP Allocation

Description: Request IPv6 address space from APNIC (Asia-Pacific Network Information Centre) including membership tiers, allocation policies, and the application process.

## APNIC Service Region

APNIC serves the Asia-Pacific region including Australia, China, Japan, India, Southeast Asia, and Pacific Island nations. IPv6 space is allocated from ranges including `2001:200::/23` and `2400::/12`.

## APNIC Membership Tiers

| Tier | Annual Fee (AUD) | Min IPv6 |
|---|---|---|
| Associate | Free | N/A (observer) |
| Micro | ~$200 | /32 |
| Small | ~$1,800 | /32 |
| Medium | ~$4,000 | /32 |
| Large | ~$15,000 | /30 |

## Application Process

```sql
Step 1: Register at https://www.apnic.net/membership/
  - Create account at MyAPNIC portal
  - Provide organization details

Step 2: Select membership tier
  - Based on number of IP addresses currently held
  - Micro tier available for small organizations

Step 3: Submit IPv6 allocation request
  - MyAPNIC → Resources → New Request
  - Choose IPv6 Direct Allocation or Assignment
  - Provide:
    * Technical justification
    * Network infrastructure description
    * Customer growth plan (for ISPs)

Step 4: APNIC review
  - Initial /32 typically auto-approved for members
  - Larger requests require staff review (2-5 days)

Step 5: Registration
  - Register allocation in APNIC Whois (APNIC Database)
  - Create RPKI ROA
  - Set up reverse DNS delegation
```

## APNIC Database Registration

```text
# Register your IPv6 allocation in APNIC Whois

inet6num:       2400:db8::/32
netname:        EXAMPLE-AP
descr:          Example Organization IPv6
country:        AU
admin-c:        ADMIN1-AP
tech-c:         TECH1-AP
mnt-by:         MAINT-AP-EXAMPLE
status:         ALLOCATED PORTABLE
source:         APNIC

# Route6 object

route6:         2400:db8::/32
descr:          Example Organization IPv6 Route
origin:         AS65001
mnt-by:         MAINT-AP-EXAMPLE
source:         APNIC
```

```bash
# Register objects via APNIC whois API
curl -X POST "https://wq.apnic.net/whois-search/apnic/route6" \
  -d "route6=2400:db8::/32&origin=AS65001&mnt-by=MAINT-AP-EXAMPLE"

# Or use APNIC's MyAPNIC portal for easier registration
```

## APNIC RPKI

```bash
# APNIC provides RPKI via hosted model (default) or delegated

# Hosted ROA via MyAPNIC:
# Login → RPKI → Create ROA
# Prefix: 2400:db8::/32
# Max length: 48
# ASN: 65001

# Validate via APNIC RPKI Validator
curl -s "https://stat.apnic.net/api/rpki-validation?resource=AS65001&prefix=2400:db8::/32"

# Verify via Routinator
routinator validate --asn 65001 --prefix 2400:db8::/32
```

## Sub-Allocation to Customers

```text
# APNIC policy for ISPs:
# Sub-allocate /48 per customer (recommended)
# Register /29 and larger sub-allocations in APNIC DB

inet6num:       2400:db8:1000::/48
netname:        CUSTOMER-A
descr:          Customer A IPv6 Assignment
country:        JP
admin-c:        CUSTADMIN-AP
tech-c:         CUSTTECH-AP
mnt-by:         MAINT-AP-EXAMPLE
status:         ASSIGNED PORTABLE
source:         APNIC
```

## Conclusion

APNIC membership provides automatic access to an initial /32 IPv6 allocation for ISPs and network operators. The Micro membership tier (AUD ~$200/year) is suitable for small organizations. Register your allocation in the APNIC Whois Database with `inet6num` and `route6` objects, then create an RPKI ROA via MyAPNIC's hosted RPKI service. Sub-allocate /48 prefixes to customers and register sub-allocations of /29 or larger. APNIC's escalating fee tiers mean large organizations pay more but receive proportionally larger address blocks.
