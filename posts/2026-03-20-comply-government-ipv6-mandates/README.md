# How to Comply with Government IPv6 Mandates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Government, Compliance, OMB, Federal Mandate, Policy

Description: Understand and comply with US government and international IPv6 deployment mandates, including OMB memoranda requirements, implementation timelines, and compliance verification approaches.

---

Multiple governments have issued IPv6 deployment mandates. The US federal government's OMB (Office of Management and Budget) mandates, along with similar directives from APNIC, RIPE, and national governments, require agencies and contractors to deploy IPv6 across their infrastructure.

## US Federal IPv6 Mandate (OMB)

```text
Key OMB IPv6 Memoranda:

1. M-05-22 (2005): "Transition Planning for Internet Protocol Version 6"
   - Required agencies to upgrade backbone infrastructure to IPv6

2. OMB M-21-07 (2020): "Completing the Transition to IPv6"
   - Updated mandate with new requirements
   - Key requirements:
     * All internet-facing federal systems: IPv6-only by FY 2025
     * All internal federal systems: IPv6-only by FY 2025
     * Federal agencies must discontinue IPv4-only procurement

3. Current status (as of 2026):
   - 80%+ of federal traffic target over IPv6
   - Agencies report progress in quarterly FISMA reports
```

## Compliance Requirements for Federal Agencies

```bash
OMB M-21-07 Requirements:

1. Public-Facing Services:
   - All public web services accessible over IPv6
   - DNS: AAAA records for all public servers
   - IPv6 HTTPS: TLS certificates valid for IPv6 addresses

   Verify:
   curl -6 https://agency.gov
   dig AAAA agency.gov +short

2. Internal Networks:
   - IPv6 required on all internal networks
   - IT infrastructure must support IPv6
   - Procurement must specify IPv6 support

3. Cloud Services:
   - Federal cloud deployments must support IPv6
   - AWS GovCloud, Azure Government: IPv6 enabled
   - CDN/WAF: Must pass IPv6 traffic
```

## Agency IPv6 Deployment Checklist

```bash
#!/bin/bash
# federal_ipv6_compliance_check.sh

# Check federal agency IPv6 compliance status

AGENCY_DOMAIN="agency.gov"

echo "=== Federal IPv6 Compliance Check for $AGENCY_DOMAIN ==="

# Check 1: AAAA record exists
echo -n "AAAA record: "
AAAA=$(dig AAAA +short $AGENCY_DOMAIN 2>/dev/null)
if [ -n "$AAAA" ]; then
    echo "PASS ($AAAA)"
else
    echo "FAIL (no AAAA record)"
fi

# Check 2: IPv6 web access
echo -n "IPv6 HTTPS access: "
if curl -6 --max-time 10 -o /dev/null -s -w "%{http_code}" \
  https://$AGENCY_DOMAIN 2>/dev/null | grep -q "200\|301\|302"; then
    echo "PASS"
else
    echo "FAIL"
fi

# Check 3: MX record resolves to IPv6
echo -n "Mail server IPv6: "
MX=$(dig +short MX $AGENCY_DOMAIN | head -1 | awk '{print $2}')
if [ -n "$MX" ] && dig +short AAAA $MX | grep -q ":"; then
    echo "PASS ($MX has IPv6)"
else
    echo "WARN (check mail server IPv6)"
fi

echo "=== Check Complete ==="
```

## Contractor IPv6 Requirements

```text
Federal contractors must comply with:

1. FISMA (Federal Information Security Modernization Act):
   - System security plans must address IPv6 security
   - IPv6 must be included in security assessments

2. FedRAMP (Cloud services):
   - Cloud providers in FedRAMP must support IPv6
   - IPv6 must be included in FedRAMP authorization package

3. Section 508 Compliance:
   - Accessibility doesn't directly require IPv6
   - But accessible web content must be IPv6-accessible

Procurement Language (example):
"The offered solution MUST support IPv6 addressing and be
capable of operating in an IPv6-only environment as defined by
NIST SP 500-267B (USGv6 Profile)."
```

## Reporting and Documentation

```text
Federal Agency IPv6 Reporting Requirements:

1. Quarterly FISMA Metrics:
   - % of internet-accessible systems with IPv6
   - % of IPv6 traffic on public-facing services
   - Report via CyberScope or successor system

2. Documentation Required:
   - IPv6 Transition Plan
   - Network topology diagrams showing IPv6
   - IPv6 security policies and procedures

3. Tracking IPv6 Deployment Progress:
   - Use NIST's IPv6 tracking dashboard
   - Agency CIO responsible for reporting
```

## International IPv6 Mandates

```text
Other Government IPv6 Requirements:

European Union:
- EU IPv6 Action Plan (2016)
- "IPv6 in European public administrations"
- Target: All EU institutions publicly accessible over IPv6

China:
- Ministry of Industry and Information Technology (MIIT)
- Requires top-1000 websites IPv6-capable
- ISPs required to provide IPv6 by mandate

India:
- National IPv6 Deployment Roadmap
- Telecom operators required to support IPv6

Australia:
- Whole-of-Government ICT Strategy includes IPv6
- ASD guidelines recommend IPv6 deployment
```

Government IPv6 mandates represent the policy driver that is accelerating enterprise IPv6 adoption, with OMB M-21-07 being the most prescriptive in requiring federal agencies to achieve IPv6-only operation for both public-facing and internal systems by specific fiscal year deadlines.
