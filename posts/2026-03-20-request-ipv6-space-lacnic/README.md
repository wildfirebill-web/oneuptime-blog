# How to Request IPv6 Address Space from LACNIC - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, LACNIC, RIR, Address Space, Latin America

Description: Request IPv6 address space from LACNIC (Latin America and Caribbean Network Information Centre) for organizations in the Latin American and Caribbean region.

## LACNIC Service Region

LACNIC serves Latin America and the Caribbean. IPv6 space is allocated from ranges including `2800::/12`. LACNIC is the smallest RIR by member count but serves a rapidly growing internet market.

## Membership and Fees

```text
LACNIC membership categories (2024, USD approximate):

Category 1: ~$400/year (small orgs, end-users)
Category 2: ~$1,500/year (medium ISPs)
Category 3: ~$3,500/year (large ISPs)
Category 4: ~$8,000/year (national ISPs)

IPv6 allocation:
  Initial /32 for ISPs (no justification required)
  /48 for end users (via sponsoring ISP)
  Subsequent allocations: show 80% utilization
```

## Application Process

```sql
1. Register at https://milacnic.lacnic.net/
   - Create organization account
   - Provide legal entity details
   - Specify country in Latin America/Caribbean

2. Select membership category
   - Based on existing IP holdings or expected usage

3. Request IPv6 resources
   - Login to milacnic.lacnic.net
   - Resources → Request → IPv6
   - Fill justification form

4. LACNIC review
   - Simple /32 requests: usually auto-approved
   - Larger requests: staff review (3-7 days)

5. Post-approval
   - Register in LACNIC Whois
   - Create RPKI ROA
   - Request reverse DNS delegation
```

## LACNIC Whois Registration

```text
# Register IPv6 allocation in LACNIC Whois Database

inet6num:       2800:db8::/32
owner:          Example ISP Brazil
country:        BR
owner-c:        ADMIN-LACNIC
tech-c:         TECH-LACNIC
inetnum:        2800:db8::/32
status:         allocated
nic-hdl-br:     ORG-EXAMPLE
source:         LACNIC

# Announce route via milacnic portal or LACNIC REST API

```

## RPKI via LACNIC

```bash
# LACNIC provides hosted RPKI for members

# Create ROA via milacnic.lacnic.net portal:
# Resources → RPKI → Manage ROAs → Create

# Or via LACNIC RPKI API
curl -X POST "https://rpki.lacnic.net/api/roas/" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "prefix": "2800:db8::/32",
    "max_length": 48,
    "asn": 65001
  }'

# Validate ROA
curl "https://rpki.lacnic.net/api/validity/AS65001/2800:db8::/32"
```

## Conclusion

LACNIC membership provides IPv6 allocations for Latin American and Caribbean organizations. The initial /32 is available immediately upon membership approval. Register resources in the LACNIC Whois Database and create RPKI ROAs to secure BGP announcements. LACNIC has Spanish and Portuguese language support for all services. Organizations outside the Latin American region cannot receive resources directly from LACNIC - they must contact their regional RIR (ARIN, RIPE NCC, APNIC, or AFRINIC).
