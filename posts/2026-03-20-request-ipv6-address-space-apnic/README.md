# How to Request IPv6 Address Space from APNIC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, APNIC, RIR, Asia Pacific, Address Space, Registry

Description: Guide to requesting IPv6 address space from APNIC for organizations in the Asia-Pacific region.

## What is APNIC?

APNIC (Asia-Pacific Network Information Centre) is the RIR responsible for IPv6 address allocation across the Asia-Pacific region, covering 56 economies from Afghanistan to Fiji.

## Membership Types

APNIC uses a tiered fee structure based on total IPv4 and IPv6 holdings:

| Category | IPv4 Holdings | Annual Fee (approx.) |
|----------|-------------|---------------------|
| Associate | None | $500 |
| Small ISP | /22-/20 | $1,500 |
| Medium ISP | /19-/18 | $4,000 |
| Large ISP | /17 and above | $8,000+ |

## Becoming an APNIC Member

### 1. Pre-registration Check

Verify your organization is in the APNIC service region and holds (or intends to obtain) an ASN.

### 2. Apply for Membership

Visit `https://www.apnic.net/membership/apply/` and complete the online form:
- Organization name and type
- Country of operation
- Technical contact details
- Existing IP resources (if any)

### 3. Sign the Membership Agreement

Agree to APNIC's membership terms which include the acceptable use policy for IP resources.

## Requesting IPv6 Address Space

### For New Members (Initial Allocation)

New members who are ISPs receive a /32 automatically:

1. Log into MyAPNIC at `myapnic.net`
2. Navigate to **Resources → IPv6 → Request IPv6**
3. Select "Initial allocation for LIR"
4. The /32 is typically approved within 1-2 business days

### For Existing Members (Additional Space)

To request a /31 or /30, demonstrate utilization of your current /32:
- Provide assignment documentation (at least 50% of sub-assignments made)
- Submit through MyAPNIC with a utilization report

### End-User Assignments

Non-ISP organizations can receive a PI /48 directly from APNIC:

```text
Request criteria:
- Demonstrate the need for globally unique IPv6 addressing
- Provide a basic network plan
- Either hold an ASN or plan to obtain one
```

## Creating Whois Objects in APNIC DB

After allocation, register your network objects:

```text
# inet6num object

inet6num: 2001:db8::/32
netname:  YOUR-NET-AP
descr:    Your ISP Name
country:  AU
admin-c:  YO1-AP
tech-c:   YO1-AP
mnt-by:   MAINT-YOUR-ORG-AP
status:   ALLOCATED PORTABLE

# route6 object
route6: 2001:db8::/32
descr: YOUR-ORG IPv6 routing
origin: AS65001
mnt-by: MAINT-YOUR-ORG-AP
```

Submit objects via the APNIC Whois portal at `wq.apnic.net` or via email to `auto-dbm@apnic.net`.

## RPKI with APNIC

APNIC provides hosted RPKI. Create ROAs through MyAPNIC:

1. Log into MyAPNIC
2. Go to **Security → RPKI → Create ROA**
3. Set prefix, maximum length, and originating ASN

```bash
# Verify your ROA is visible
rpki-client -v -r /var/cache/rpki-client
# Or use an online RPKI validator
curl https://rpki-validator.apnic.net/api/v1/validity/AS65001/2001:db8::/32
```

## Conclusion

APNIC membership provides ISPs in the Asia-Pacific region with automatic /32 IPv6 allocations. The process through MyAPNIC is streamlined, and RPKI support is built into the portal for immediate route security configuration.
