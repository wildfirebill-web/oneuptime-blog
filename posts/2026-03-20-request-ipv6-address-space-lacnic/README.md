# How to Request IPv6 Address Space from LACNIC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, LACNIC, RIR, Latin America, Address Space, Registry

Description: Guide to requesting IPv6 address space from LACNIC for organizations in Latin America and the Caribbean.

## What is LACNIC?

LACNIC (Latin America and Caribbean Network Information Centre) is the RIR for Latin America and the Caribbean, serving 33 countries. It manages the allocation of IPv6 address space in the region.

## Who Can Request from LACNIC?

- **Internet Service Providers (ISPs)**: Organizations providing connectivity services
- **End-User Organizations**: Enterprises and institutions with documented multi-homing needs
- **NIRs**: National Internet Registries that sub-delegate to local ISPs

## Membership and Fees

LACNIC uses a category-based membership fee:

| Category | Description | Annual Fee (approx.) |
|----------|------------|---------------------|
| Extra Small | Micro-ISPs | $500 USD |
| Small | Regional ISPs | $1,200 USD |
| Medium | National ISPs | $4,000 USD |
| Large | Major operators | $10,000+ USD |

## Step-by-Step Request Process

### 1. Create a LACNIC Account

Register at `https://lacnic.net/registro/` with:
- Organization details
- ASN (or request one simultaneously)
- Technical and administrative contact information

### 2. Submit an IPv6 Request

Log into the LACNIC member portal and navigate to the IPv6 request form:

```text
Required information:
- Organization type (ISP, end-user, NIR)
- Requested prefix size (/32 for ISPs, /48 for end-users)
- Network plan description
- Number of current customers (for ISPs)
- Country of operation
```

### 3. Initial Allocation Policy

LACNIC's policy for initial ISP allocations:
- ISPs with existing LACNIC IPv4 resources receive /32 IPv6
- New entrants may receive /32 with a deployment plan

## Delegating to Customers

After receiving your /32, create customer delegation objects in the LACNIC Whois database:

```text
# Customer /48 assignment object

inet6num: 2001:db8:100::/48
owner:    CUSTOMER-A LTDA
country:  BR
owner-c:  CAL
tech-c:   CAL
inetnum:  2001:db8::/32
status:   allocated
nic-hdl:  CAL
```

## RPKI Configuration with LACNIC

LACNIC provides hosted RPKI services. After allocation:

1. Access the RPKI portal through your LACNIC member account
2. Request a resource certificate for your prefix
3. Create ROA entries for all announced prefixes

```bash
# Verify ROA using LACNIC's RPKI validator
curl -s "https://rpki.lacnic.net/rpki-validator/api/v1/validity/AS65001/2001:db8::/32" \
  | python3 -m json.tool
```

## Policy for IPv6 in Brazil (BRN/CGI.br)

In Brazil, LACNIC resources are managed through CGI.br (Brazilian Internet Steering Committee). Brazilian organizations may interact with both LACNIC and CGI.br. Contact `registro.br` for Brazil-specific guidance.

## Contact and Support

- Portal: `https://lacnic.net`
- Email: `hostmaster@lacnic.net`
- Phone support available in Spanish and Portuguese

## Conclusion

LACNIC serves as the IPv6 address registry for Latin America and the Caribbean. ISPs in the region receive /32 allocations, and the process through the LACNIC member portal is available in Spanish, Portuguese, and English. Creating RPKI ROAs immediately after allocation protects against route hijacking.
