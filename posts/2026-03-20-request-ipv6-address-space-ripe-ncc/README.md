# How to Request IPv6 Address Space from RIPE NCC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, RIPE NCC, RIR, Address Space, Europe, Registry

Description: Step-by-step guide to requesting IPv6 address space from RIPE NCC for organizations in Europe, the Middle East, and Central Asia.

## What is RIPE NCC?

RIPE NCC (Réseaux IP Européens Network Coordination Centre) is the RIR responsible for IPv6 address allocation in Europe, the Middle East, and parts of Central Asia. It serves over 25,000 members across 75+ countries.

## RIPE NCC Membership

Unlike ARIN, RIPE NCC uses a membership model. To receive IPv6 address space directly from RIPE NCC, you must become a member (LIR - Local Internet Registry).

Alternatively, end-users can receive IPv6 space from an LIR (their ISP) without becoming members.

## Becoming an LIR (ISPs and Large Organizations)

### 1. Register as an LIR

Visit `https://lirportal.ripe.net` and complete the LIR registration:
- Company registration documents
- Proof of address
- Identification of the account holder
- Payment of the membership fee (~€1,400/year for a Standard LIR)

### 2. IPv6 Allocation for LIRs

All new LIRs receive a /32 IPv6 allocation automatically upon becoming members. No separate justification is needed for the initial /32.

The allocation is created in the RIPE database as:

```text
inet6num: 2001:db8::/32
netname:  YOUR-NET-NAME
descr:    Your Organization Name
country:  DE
admin-c:  YNO1-RIPE
tech-c:   YNO1-RIPE
status:   ALLOCATED-BY-RIR
mnt-by:   RIPE-NCC-HM-MNT
mnt-lower: YOUR-MNT
```

### 3. Additional Allocations

After deploying your initial /32, you can request additional space by demonstrating usage:
- Slow start: /32 → /31 → /30 based on utilization
- Show at least 50% utilization of current allocation

## End-User Assignments (Non-ISPs)

If your organization is not an ISP and does not want LIR membership:

1. Request a PI (Provider Independent) /48 through your current ISP/LIR
2. Alternatively, become a member and request a /48 directly

## Creating RIPE Database Objects

After receiving your allocation, create the necessary RIPE database objects:

```text
# Create inet6num objects for customer assignments

inet6num: 2001:db8:1::/48
netname:  CUSTOMER-A-NET
descr:    Customer A IPv6 Block
country:  NL
admin-c:  CA1-RIPE
tech-c:   CA1-RIPE
status:   ASSIGNED
mnt-by:   YOUR-MNT

# Create route6 object for BGP routing
route6: 2001:db8::/32
descr: YOUR-ORG IPv6 Route
origin: AS65001
mnt-by: YOUR-MNT
```

## RPKI ROA Creation

Create a ROA in the RIPE NCC RPKI portal to secure your route announcements:

1. Log in to `lirportal.ripe.net`
2. Navigate to **Resource Certification (RPKI)**
3. Create a ROA with your ASN, prefix, and maximum prefix length

## Whois Queries

Verify your allocation in the RIPE database:

```bash
# Query RIPE Whois
whois -h whois.ripe.net 2001:db8::/32

# Query for route objects
whois -h whois.ripe.net -T route6 2001:db8::/32
```

## Conclusion

RIPE NCC provides IPv6 allocations to member LIRs automatically upon joining. The /32 initial allocation, combined with the RIPE database's hierarchical object model and RPKI support, makes it straightforward to deploy and secure IPv6 for European organizations.
