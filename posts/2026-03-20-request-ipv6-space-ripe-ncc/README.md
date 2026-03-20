# How to Request IPv6 Address Space from RIPE NCC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, RIPE NCC, RIR, Address Space, IP Allocation

Description: Request IPv6 address space from RIPE NCC as an LIR or end-user organization including membership, allocation policies, and the RIPE NCC application process.

## RIPE NCC Service Region

RIPE NCC serves Europe, the Middle East, and parts of Central Asia. IPv6 address space comes from the `2001::/32` and `2a00::/11` ranges assigned to RIPE NCC.

## LIR vs End-User

| Type | Who | Min Prefix | Fee |
|---|---|---|---|
| LIR (Local Internet Registry) | ISPs, large orgs | /32 | ~€1,400/year |
| End-user assignment | Enterprises | /48 | Via LIR sponsorship |

End-users cannot request directly from RIPE NCC — they must go through a sponsoring LIR.

## Becoming an LIR

```
Requirements to become a RIPE NCC LIR:

1. Organization must be legally established in RIPE service region
2. Must have technical competency to manage IP resources
3. Sign RIPE NCC Standard Service Agreement
4. Pay LIR membership fee (~€1,400/year in 2024)

Process:
1. Go to https://www.ripe.net/membership/
2. Fill out LIR application form
3. Provide:
   - Legal entity documentation
   - Proof of address in service region
   - Technical justification for IP space
4. RIPE NCC account manager reviews (1-2 weeks)
5. Upon approval: receive IPv6 /32 allocation
```

## Initial IPv6 /32 Allocation

```
RIPE NCC policy (2023):

Every LIR receives ONE /32 allocation upon membership.
No additional justification required for first /32.
Subsequent allocations require:
  - 80% utilization of current allocation
  - Documentation of assignment plan

Your /32 allows:
  - 65,536 /48 assignments to customers
  - Or any combination of /48 to /128 assignments
```

## RIPE NCC Database: Register Your Allocation

```
# After receiving allocation, register in RIPE Database

# Example RIPE Database objects

# inet6num object (your allocation)
inet6num:       2a01:db8::/32
netname:        EXAMPLE-NET
descr:          Example ISP IPv6 Space
country:        NL
admin-c:        ADMIN1-RIPE
tech-c:         TECH1-RIPE
status:         ALLOCATED-BY-RIR
mnt-by:         EXAMPLE-MNT
mnt-routes:     EXAMPLE-MNT
source:         RIPE

# route6 object (BGP announcement)
route6:         2a01:db8::/32
descr:          Example ISP IPv6
origin:         AS65001
mnt-by:         EXAMPLE-MNT
source:         RIPE
```

```bash
# Create route6 object via RIPE NCC API (Syncronaut/REST)
curl -X POST "https://rest.db.ripe.net/ripe/route6" \
  -H "Content-Type: application/json" \
  -d '{
    "objects": {
      "object": [{
        "type": "route6",
        "attributes": {
          "attribute": [
            {"name": "route6",   "value": "2a01:db8::/32"},
            {"name": "origin",   "value": "AS65001"},
            {"name": "mnt-by",   "value": "EXAMPLE-MNT"},
            {"name": "source",   "value": "RIPE"}
          ]
        }
      }]
    }
  }' \
  --user "EXAMPLE-MNT:password"
```

## RPKI ROA via RIPE NCC

```bash
# Create ROA via RIPE NCC dashboard
# https://my.ripe.net → RPKI → Create ROA

# Or via RIPE NCC API
curl -X POST "https://my.ripe.net/api/whois/rpki/roas" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prefix": "2a01:db8::/32",
    "max-length": 48,
    "asn": "AS65001"
  }'

# Verify ROA
curl -s "https://stat.ripe.net/data/rpki-validation/data.json?resource=AS65001&prefix=2a01:db8::/32"
```

## Sub-Allocating to Customers

```
# RIPE policy: document assignments in RIPE Database

# Customer assignment example:
inet6num:       2a01:db8:1000::/48
netname:        CUSTOMER-A-IPV6
descr:          Customer A Assignment
country:        DE
admin-c:        CUSTADMIN-RIPE
tech-c:         CUSTTECH-RIPE
status:         ASSIGNED PA
mnt-by:         EXAMPLE-MNT
mnt-lower:      CUSTOMER-A-MNT
source:         RIPE

# Assignments of /48 and larger must be registered in RIPE DB within 30 days
```

## Conclusion

RIPE NCC membership grants LIRs an initial /32 IPv6 allocation automatically — no additional justification required for the first allocation. Register your allocation and route6 objects in the RIPE Database using `inet6num` and `route6` object types. Create an RPKI ROA via the RIPE NCC dashboard to authorize BGP announcements and prevent route hijacking. When sub-allocating /48s to customers, document each assignment as an `inet6num` object with status `ASSIGNED PA` within 30 days. Subsequent /32 allocations require demonstrating 80% utilization of your current space.
