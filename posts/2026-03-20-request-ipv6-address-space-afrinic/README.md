# How to Request IPv6 Address Space from AFRINIC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, AFRINIC, RIR, Africa, Address Space, Registry

Description: Guide to requesting IPv6 address space from AFRINIC for organizations operating in Africa.

## What is AFRINIC?

AFRINIC (African Network Information Centre) is the RIR responsible for IPv6 address allocation across the African continent, including 54 countries. It is headquartered in Mauritius.

## AFRINIC Service Region

AFRINIC covers all countries in Africa. For island territories in the Indian Ocean and Atlantic Ocean associated with Africa, AFRINIC is also the relevant RIR.

## Membership Categories

| Category | Description | Annual Fee (approx.) |
|----------|------------|---------------------|
| Extra-Small | New/micro ISPs | $500 USD |
| Small | Regional ISPs | $1,000 USD |
| Medium | National operators | $3,000 USD |
| Large | Major carriers | $8,000+ USD |
| Associate | Non-ISP organizations | $400 USD |

## How to Apply

### 1. Register on AFRINIC Online

Create an account at `https://my.afrinic.net`:
- Organization name and registration number
- Physical address in Africa
- Technical and administrative POC details

### 2. Submit IPv6 Request

Navigate to **Resources → Request IPv6 Address Space**:

```
Required documentation:
- Description of network topology
- Current number of customers or end-sites
- Deployment plan for IPv6 (timeline, milestones)
- Existing IPv4 resources (if any)
- ASN number (or simultaneous ASN request)
```

### 3. Initial Allocation Sizes

- ISPs with existing AFRINIC IPv4 resources: /32
- New ISPs with deployment plan: /32
- End-user organizations: /48

## Creating Whois Objects

After allocation, register your objects in the AFRINIC Whois database:

```
# inet6num object
inet6num: 2001:db8::/32
netname:  YOUR-NET-AF
descr:    Your ISP Name - Africa
country:  NG
admin-c:  YNO-AFRINIC
tech-c:   YNO-AFRINIC
mnt-by:   AFRINIC-HM-MNT
mnt-lower: YOUR-MNT-AFRINIC
status:   ALLOCATED-BY-RIR

# route6 announcement
route6: 2001:db8::/32
descr: YOUR-ORG primary route
origin: AS65001
mnt-by: YOUR-MNT-AFRINIC
```

Submit via the AFRINIC Whois portal or email `hostmaster@afrinic.net`.

## RPKI with AFRINIC

AFRINIC provides hosted RPKI through their member portal:

1. Log into My.AFRINIC.NET
2. Navigate to **RPKI → My Resource Certificates**
3. Create ROA for your prefix and ASN

## IPv6 Adoption Context in Africa

Africa has significant IPv6 adoption momentum driven by:
- Mobile networks (major carriers deploying IPv6-only LTE)
- IPv4 exhaustion at AFRINIC (exhausted in 2021)
- IXP growth in Nairobi, Lagos, Johannesburg, and Cairo

Organizations in Africa should prioritize IPv6 deployment as IPv4 addresses are now only available through the waiting list policy.

## Support Resources

- Portal: `https://my.afrinic.net`
- Email: `hostmaster@afrinic.net`
- AFRINIC Academy: Free IPv6 training resources
- Community: `https://lists.afrinic.net`

## Conclusion

AFRINIC is the IPv6 registry for Africa, and with IPv4 exhaustion already complete in the region, IPv6 deployment is increasingly urgent. The membership and request process is straightforward through the My.AFRINIC.NET portal, with support available in English and French.
