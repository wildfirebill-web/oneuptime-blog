# How to Measure IPv6 Migration Progress

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Migration, Metrics, KPIs, Progress Tracking

Description: Define and measure key performance indicators for IPv6 migration progress including AAAA record coverage, IPv6 traffic percentage, and service readiness scores.

## Introduction

Measuring IPv6 migration progress requires quantitative metrics that track actual enablement rather than subjective assessment. Good metrics include the percentage of services with AAAA records, the share of production traffic carried over IPv6, the number of code repositories free of IPv4 hardcoding, and the percentage of infrastructure with IPv6 addresses.

## Key Metrics

### 1. AAAA Record Coverage

```python
#!/usr/bin/env python3
# measure_aaaa_coverage.py

import dns.resolver
from typing import NamedTuple

class ServiceStatus(NamedTuple):
    hostname: str
    has_a: bool
    has_aaaa: bool

# All services that should have AAAA records
ALL_SERVICES = [
    "www.example.com",
    "api.example.com",
    "mail.example.com",
    "vpn.example.com",
    "auth.example.com",
    "docs.example.com",
    "cdn.example.com",
]

def check_dns(hostname: str) -> ServiceStatus:
    has_a = has_aaaa = False
    for record_type, flag in [('A', True), ('AAAA', False)]:
        try:
            dns.resolver.resolve(hostname, record_type)
            if record_type == 'A':
                has_a = True
            else:
                has_aaaa = True
        except:
            pass
    return ServiceStatus(hostname, has_a, has_aaaa)

results = [check_dns(s) for s in ALL_SERVICES]
coverage = sum(1 for r in results if r.has_aaaa) / len(results) * 100

print(f"AAAA Coverage: {coverage:.1f}% ({sum(1 for r in results if r.has_aaaa)}/{len(results)})")
for r in results:
    status = "OK" if r.has_aaaa else "MISSING"
    print(f"  [{status}] {r.hostname}")
```

### 2. IPv6 Traffic Percentage

```promql
# Prometheus query: IPv6 traffic as % of total
sum(rate(http_requests_total{ip_version="ipv6"}[1h]))
/
sum(rate(http_requests_total[1h]))
* 100
```

Track this metric weekly and trend toward the goal of 40%+ within 6 months of full AAAA publication.

### 3. Infrastructure IPv6 Readiness

```bash
#!/bin/bash
# measure_infra_readiness.sh

TOTAL=0
IPV6_READY=0

# Check Kubernetes nodes
while IFS= read -r node; do
    TOTAL=$((TOTAL+1))
    # Check if node has IPv6 address
    if kubectl get node "$node" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}' | grep -q ":"; then
        IPV6_READY=$((IPV6_READY+1))
    fi
done < <(kubectl get nodes -o name | sed 's/node\///')

echo "Kubernetes nodes with IPv6: $IPV6_READY/$TOTAL"

# Check services with dual-stack
TOTAL_SVC=$(kubectl get services --all-namespaces --no-headers | wc -l)
DUAL_STACK=$(kubectl get services --all-namespaces -o json | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
count = sum(1 for svc in data['items']
            if svc['spec'].get('ipFamilyPolicy') in ('PreferDualStack', 'RequireDualStack'))
print(count)
")
echo "Dual-stack Kubernetes services: $DUAL_STACK/$TOTAL_SVC"
```

### 4. Application Readiness Score

```python
#!/usr/bin/env python3
# score_app_readiness.py

import subprocess
import os
from pathlib import Path

def check_repo(repo_path: str) -> dict:
    """Score an application repository for IPv6 readiness."""
    score = 0
    max_score = 100
    issues = []

    # Check 1: No hardcoded IPv4 addresses (20 points)
    result = subprocess.run(
        ["grep", "-r", "-E", r"\b([0-9]{1,3}\.){3}[0-9]{1,3}\b",
         "--include=*.py", "--include=*.go", "--include=*.js",
         repo_path],
        capture_output=True, text=True
    )
    ipv4_count = len(result.stdout.splitlines())
    if ipv4_count == 0:
        score += 20
    else:
        issues.append(f"Hardcoded IPv4 addresses: {ipv4_count}")

    # Check 2: Uses :: binding (not 0.0.0.0) (20 points)
    result = subprocess.run(
        ["grep", "-r", "0.0.0.0", "--include=*.py", "--include=*.go",
         "--include=*.yaml", "--include=*.env", repo_path],
        capture_output=True, text=True
    )
    if len(result.stdout.splitlines()) == 0:
        score += 20
    else:
        issues.append(f"Found 0.0.0.0 bindings")

    # Check 3: Tests exist for IPv6 (20 points)
    has_ipv6_tests = any(
        "ipv6" in f.read_text().lower()
        for f in Path(repo_path).rglob("test_*.py")
        if f.stat().st_size < 100000  # Skip large files
    )
    if has_ipv6_tests:
        score += 20
    else:
        issues.append("No IPv6 tests found")

    # Check 4: No AF_INET without AF_INET6 option (20 points)
    result = subprocess.run(
        ["grep", "-r", r"AF_INET\b", "--include=*.py", repo_path],
        capture_output=True, text=True
    )
    if len(result.stdout.splitlines()) == 0:
        score += 20
    else:
        issues.append("AF_INET (non-IPv6 socket) found")

    # Check 5: Docker/K8s config uses :: (20 points)
    for fname in ["docker-compose.yml", "Dockerfile", "values.yaml"]:
        fpath = os.path.join(repo_path, fname)
        if os.path.exists(fpath):
            content = open(fpath).read()
            if "::" in content or "ipv6" in content.lower():
                score += 20
                break
    else:
        issues.append("No IPv6 config in Docker/K8s files")

    return {"score": score, "max": max_score, "issues": issues}

result = check_repo(".")
print(f"IPv6 Readiness Score: {result['score']}/{result['max']}")
if result['issues']:
    print("Issues:")
    for issue in result['issues']:
        print(f"  - {issue}")
```

## Progress Dashboard Template

| Metric | Baseline | Week 4 | Week 8 | Week 12 | Target |
|--------|----------|--------|--------|---------|--------|
| AAAA record coverage | 0% | 20% | 60% | 100% | 100% |
| IPv6 traffic % | 0% | 2% | 15% | 40% | 40% |
| Services on dual-stack | 0 | 5 | 20 | All | All |
| App readiness score | 40/100 | 60/100 | 80/100 | 95/100 | 90/100 |
| IPv4 hardcoding issues | 48 | 30 | 10 | 2 | 0 |

## Conclusion

IPv6 migration progress measurement uses four primary metrics: AAAA DNS record coverage (percentage of services reachable via IPv6), production IPv6 traffic percentage (business outcome metric), infrastructure readiness (devices and services with IPv6 configured), and application readiness score (code quality metric). Report these weekly during the migration phase in a simple dashboard. The IPv6 traffic percentage is the most meaningful outcome metric — it directly reflects how many real users benefit from IPv6 enablement.
