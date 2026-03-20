# How to Request IPv6 Address Space from ARIN

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ARIN, RIR, Address Space, ISP, Registry

Description: A step-by-step guide to requesting IPv6 address space from ARIN (American Registry for Internet Numbers) for ISPs and end-users.

## What is ARIN?

ARIN (American Registry for Internet Numbers) manages IPv6 address allocation for the United States, Canada, and many Caribbean and North Atlantic territories. Organizations in this region request IPv6 space through ARIN.

## Types of IPv6 Allocations from ARIN

- **ISP Allocation**: Large blocks (/32) for ISPs to delegate to customers
- **End-User Assignment**: Direct assignments (/48) for organizations that do not provide services to others
- **Critical Infrastructure**: Special allocations for DNS root servers, IXPs

## Prerequisites

Before applying, you need:
1. An ARIN Online account (create at account.arin.net)
2. A signed RSA (Registration Services Agreement) with ARIN
3. Justification documentation

## Step-by-Step ISP Allocation Request

### 1. Create an ARIN Online Account

Go to `https://account.arin.net` and register your organization. You will need:
- Organization legal name
- Physical address
- Technical and administrative POC contact details

### 2. Sign the RSA

The RSA is a legal agreement covering IP address usage policies. Sign it electronically through the ARIN portal.

### 3. Submit a Request for IPv6 Space

For ISPs, navigate to: **ARIN Online → IPv6 → Request IPv6 Address Space**

Fill in:
- **Organization**: Your registered organization
- **Prefix Size**: Typically /32 for ISPs with existing IPv4 allocations
- **Justification**: Describe your network and customer base
- **Current IPv6 Use**: Note any existing IPv6 deployments

### 4. Provide Justification

ARIN requires justification for initial allocations. For ISPs, you typically need to demonstrate:

```text
- Number of existing customers (home, business)
- Planned IPv6 deployment timeline
- Current infrastructure (ASN, IPv4 allocation size)
- Network diagram showing IPv6 deployment plan
```

### 5. Initial Allocation Sizes

| Organization Type | Minimum | Typical |
|------------------|---------|---------|
| ISP with /20 IPv4 | /32 | /32 |
| Large ISP | /32 | /28 or /24 |
| End-user org | /48 | /48 |

## After Approval

Once approved, ARIN will:
1. Allocate your prefix in the ARIN database (Whois)
2. Send confirmation to your technical POC
3. The allocation is published in the global routing table after you announce it via BGP

## Register Your Routes (ROA)

After receiving your prefix, immediately create a Route Origin Authorization (ROA) in ARIN's RPKI system:

```text
In ARIN Online:
1. Go to Routing Security → Route Origin Authorizations
2. Create New ROA
3. Set your ASN and prefix
4. Set max prefix length (typically same as allocated prefix)
```

This protects against BGP route hijacking.

## Fees

ARIN charges annual registration fees based on allocation size. Check `arin.net/fees` for current pricing. ISPs pay based on total IPv4 and IPv6 holdings.

## Conclusion

Requesting IPv6 space from ARIN involves creating an account, signing the RSA, providing deployment justification, and creating ROAs for route security. The process typically takes 1-5 business days for straightforward ISP requests.
