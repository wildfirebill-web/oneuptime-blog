# How to Request IPv6 Address Space from AFRINIC - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, AFRINIC, RIR, Address Space, Africa

Description: Request IPv6 address space from AFRINIC (African Network Information Centre) for organizations in Africa including membership requirements and application process.

## AFRINIC Service Region

AFRINIC serves the African continent and Indian Ocean region. IPv6 space is allocated from `2c00::/12`. AFRINIC is the newest RIR and actively promotes IPv6 adoption across Africa.

## Membership Categories

```text
AFRINIC membership (2024, USD approximate):

Micro ISP:     ~$100/year (small, <500 customers)
Small ISP:     ~$500/year
Medium ISP:    ~$1,500/year
Large ISP:     ~$3,500/year
Extra Large:   ~$10,000/year

IPv6 allocation:
  Initial /32 for all ISP members
  No additional justification for first /32
  End-users: /48 via sponsoring LIR
```

## Application Process

```sql
1. Register at https://my.afrinic.net/
   - Create account
   - Submit organization details
   - Provide proof of legal entity in Africa

2. AFRINIC membership activation
   - Staff verifies eligibility
   - Usually 1-2 weeks

3. Request IPv6 space
   - Login to MyAFRINIC
   - Resources → Request IPv6 Address Space
   - Select allocation size (/32 for ISPs)

4. Receive allocation
   - Typically from 2c00::/12 range
   - AFRINIC may assign from sub-ranges

5. Post-allocation requirements
   - Register in AFRINIC Whois Database
   - Create RPKI ROA (AFRINIC supports hosted RPKI)
   - Set up reverse DNS
```

## AFRINIC Whois Registration

```text
inet6num:       2c0f:db8::/32
netname:        EXAMPLE-ZA-IPV6
descr:          Example ISP South Africa IPv6
country:        ZA
admin-c:        ADMIN-AFRINIC
tech-c:         TECH-AFRINIC
mnt-by:         MAINT-EXAMPLE-ZA
status:         ALLOCATED PA
source:         AFRINIC
```

## RPKI via AFRINIC

```bash
# AFRINIC hosted RPKI via MyAFRINIC portal

# https://my.afrinic.net → Resources → RPKI

# Create ROA:
# Prefix: 2c0f:db8::/32
# Max Length: 48
# ASN: your AS number

# Verify via AFRINIC RPKI Validator
curl "https://rpki.afrinic.net/api/v1/validity/?asn=AS65001&prefix=2c0f:db8::/32"
```

## Conclusion

AFRINIC is actively working to increase IPv6 adoption across Africa with competitive membership fees including a Micro ISP tier at ~$100/year. The initial /32 IPv6 allocation is available to all ISP members. AFRINIC provides training and technical assistance for new members implementing IPv6, including remote support in English and French. Register resources in AFRINIC Whois and create RPKI ROAs to complete the IPv6 deployment process.
