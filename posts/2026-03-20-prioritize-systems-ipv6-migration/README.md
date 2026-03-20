# How to Prioritize Systems for IPv6 Migration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Migration, Prioritization, Planning, Network Strategy

Description: Create a prioritization framework for IPv6 migration across systems and services, balancing business risk, technical complexity, and strategic value.

## Introduction

Migrating all systems to IPv6 simultaneously is impractical. A prioritization framework helps sequence migration based on strategic impact, risk, and effort. Systems that provide IPv6 to many others (DNS, load balancers) should be migrated early; systems with high risk and complex change should come later after the team has practice.

## Prioritization Criteria

Score each system on these dimensions (1-5 scale):

| Criterion | Description |
|-----------|-------------|
| **Business Value** | How much does IPv6 on this system benefit users/business? |
| **Upstream Dependency** | How many other systems depend on this for IPv6? |
| **Effort to Migrate** | How complex is the change? (5 = easy, 1 = very hard) |
| **Risk if Done Late** | What is the risk of delaying this system's migration? |
| **External Facing** | Is this accessible to external IPv6 users? |

## Prioritization Scoring Tool

```python
#!/usr/bin/env python3
# ipv6_prioritization.py

from dataclasses import dataclass, field
from typing import List

@dataclass
class System:
    name: str
    business_value: int    # 1-5
    upstream_dependency: int  # 1-5 (5 = many systems depend on it)
    ease_of_migration: int    # 1-5 (5 = easiest)
    risk_if_delayed: int      # 1-5
    external_facing: int      # 1 or 5

    @property
    def priority_score(self) -> float:
        # Weighted formula
        return (
            self.business_value * 0.25 +
            self.upstream_dependency * 0.30 +
            self.ease_of_migration * 0.15 +
            self.risk_if_delayed * 0.20 +
            self.external_facing * 0.10
        )

systems = [
    System("DNS resolvers",        5, 5, 4, 5, 3),
    System("Core routers",         4, 5, 3, 5, 2),
    System("Internet firewall",    4, 4, 3, 4, 4),
    System("Web load balancer",    5, 3, 4, 4, 5),
    System("Public website",       5, 2, 4, 3, 5),
    System("API gateway",          4, 3, 3, 3, 5),
    System("Monitoring (Prometheus)", 3, 2, 4, 4, 2),
    System("CI/CD pipelines",      3, 1, 3, 2, 1),
    System("Internal CRM",         2, 1, 2, 2, 1),
    System("Legacy billing system",1, 1, 1, 1, 1),
]

print(f"{'System':<35} {'Score':>6} {'Priority'}")
print("-" * 60)
sorted_systems = sorted(systems, key=lambda s: s.priority_score, reverse=True)
for i, s in enumerate(sorted_systems):
    tier = "Wave 1" if i < 3 else "Wave 2" if i < 7 else "Wave 3"
    print(f"{s.name:<35} {s.priority_score:>6.2f}  {tier}")
```

## Migration Wave Framework

### Wave 1: Foundation (High Impact, Lower Risk)

These systems must be migrated first as they enable everything else:

| System | Reason |
|--------|--------|
| DNS resolvers | AAAA record resolution — without this, nothing else works |
| Core routers | Carry IPv6 traffic between all other systems |
| Internet firewall | Must allow IPv6 before any external-facing service goes live |
| Network monitoring | Visibility before rollout; catch issues early |

### Wave 2: Internet-Facing Services (High Value)

Visible to external IPv6 users and generate business value:

| System | Migration Action |
|--------|-----------------|
| Web servers / CDN | Add AAAA to DNS; configure load balancer IPv6 VIPs |
| API gateways | Enable IPv6 listener; update TLS certificates if needed |
| Mail servers | Add AAAA to MX/SPF; configure IPv6 SMTP |
| VPN endpoints | Enable IKEv2 with IPv6 for remote users |

### Wave 3: Internal Services

Lower urgency but necessary for full IPv6 operation:

| System | Migration Action |
|--------|-----------------|
| Internal applications | Fix socket binding; remove IPv4 hardcoding |
| Databases | Enable IPv6 listener; update connection strings |
| CI/CD pipelines | Enable IPv6 in build networks |
| IPAM/NMS | Enable IPv6 management |

### Wave 4: Legacy Systems

Complex or high-risk changes; tackle after team has practice:

| System | Strategy |
|--------|---------|
| Legacy billing/ERP | NAT64 proxy if vendor can't update |
| Hardware-based systems | Contact vendor for IPv6 roadmap |
| Outsourced/SaaS | Vendor dependency — engage early |

## Decision Matrix

```
                    HIGH VALUE
                    ^
Wave 1              |              Wave 2
(Infra/Foundation)  |           (Internet-facing)
                    |
LOW EFFORT <--------+--------> HIGH EFFORT
                    |
Wave 3              |              Wave 4
(Internal services) |           (Legacy/complex)
                    |
                    LOW VALUE
```

## Conclusion

Prioritize IPv6 migration by starting with systems that unblock everything else (DNS, core routers, firewalls), then move to external-facing services that deliver business value, followed by internal services, and finally legacy or complex systems. Use the priority score formula to rank within each wave when resources are constrained. A phased wave approach limits blast radius — an issue in Wave 2 does not affect the foundation infrastructure enabled in Wave 1.
