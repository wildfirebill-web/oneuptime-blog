# How to Automate IPv6 Change Management Workflows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Change Management, Automation, Git, CI/CD, Ansible

Description: Implement automated IPv6 change management workflows with Git-based approval processes, pre-change validation, automated deployment, and rollback capabilities.

## Introduction

Manual change management for IPv6 configurations is error-prone and slow. A Git-based workflow with automated validation, staged deployment, and automatic rollback provides consistency and auditability. Every change is reviewed, tested, and logged.

## Workflow Overview

```text
1. Engineer creates change in Git branch
2. Pre-commit hooks validate IPv6 syntax and policy
3. Pull request triggers CI validation pipeline
4. Automated tests run against lab topology
5. Peer review required before merge
6. Merge triggers automated deployment
7. Post-deployment health checks verify success
8. Automatic rollback if health checks fail
```

## Change Definition Format

```yaml
# changes/2026-03-20-add-r1-peering.yml

---
change_id: CHG-2026-031
description: "Add IPv6 BGP peering between R1 and R4"
risk_level: medium
rollback_plan: "Remove neighbor 5f00:fe81:4:: from R1 BGP config"
pre_checks:
  - "ping6 5f00:fe81:4:: from R1"
  - "verify R4 is reachable via IS-IS"
post_checks:
  - "bgp_neighbor_state R1 5f00:fe81:4:: established"
  - "bgp_prefixes_received R1 5f00:fe81:4:: > 0"
devices:
  - name: R1
    config: |
      router bgp 65001
        neighbor 5f00:fe81:4:: remote-as 65001
        neighbor 5f00:fe81:4:: update-source Loopback0
      !
```

## Pre-Change Validation Script

```python
#!/usr/bin/env python3
"""Validate IPv6 change before deployment."""
import ipaddress
import yaml
import sys
import re

def validate_change_file(change_file: str) -> list:
    """Return list of validation errors."""
    errors = []

    with open(change_file) as f:
        change = yaml.safe_load(f)

    # Validate required fields
    for field in ["change_id", "description", "risk_level", "devices"]:
        if field not in change:
            errors.append(f"Missing required field: {field}")

    # Validate all IPv6 addresses in config blocks
    for device in change.get("devices", []):
        config = device.get("config", "")
        # Extract potential IPv6 addresses
        ipv6_pattern = re.compile(
            r'(?:[0-9a-fA-F]{0,4}:){2,7}[0-9a-fA-F]{0,4}(?:/\d+)?'
        )
        for match in ipv6_pattern.finditer(config):
            addr_str = match.group().rstrip("/")
            try:
                ipaddress.ip_address(addr_str)
            except ValueError:
                # Might be a prefix
                try:
                    ipaddress.ip_network(match.group(), strict=False)
                except ValueError:
                    errors.append(f"Invalid IPv6 address in config: {match.group()}")

    return errors

if __name__ == "__main__":
    change_file = sys.argv[1]
    errors = validate_change_file(change_file)
    if errors:
        for e in errors:
            print(f"ERROR: {e}")
        sys.exit(1)
    print("Validation passed")
```

## GitHub Actions CI Pipeline

```yaml
# .github/workflows/ipv6-changes.yml
name: IPv6 Change Validation

on:
  pull_request:
    paths:
      - 'changes/**/*.yml'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install napalm netmiko pyyaml jinja2

      - name: Validate change files
        run: |
          for f in changes/*.yml; do
            echo "Validating $f"
            python scripts/validate_change.py "$f"
          done

      - name: Run lab tests
        run: |
          python scripts/deploy_to_lab.py --dry-run changes/*.yml

  deploy:
    needs: validate
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy and verify
        run: python scripts/deploy_with_rollback.py
        env:
          NETWORK_PASSWORD: ${{ secrets.NETWORK_PASSWORD }}
```

## Deployment with Automatic Rollback

```python
import subprocess
import time

def deploy_with_rollback(change_file: str, device) -> bool:
    """Deploy a change with automatic rollback on failure."""
    with open(change_file) as f:
        import yaml
        change = yaml.safe_load(f)

    # Take pre-change snapshot
    snapshot = device.get_config()

    # Apply change
    try:
        for dev_config in change["devices"]:
            device.load_merge_candidate(config=dev_config["config"])
            device.commit_config()
    except Exception as e:
        print(f"Deployment failed: {e}")
        device.discard_config()
        return False

    # Wait for convergence
    time.sleep(30)

    # Run post-checks
    for check in change.get("post_checks", []):
        if not run_health_check(device, check):
            print(f"Post-check failed: {check}")
            print("Rolling back...")
            device.load_replace_candidate(config=snapshot["running"])
            device.commit_config()
            return False

    return True
```

## Conclusion

Git-based change management with automated validation and rollback brings software development best practices to IPv6 network operations. Every change is peer-reviewed, validated, and deployed consistently. Automatic rollback prevents outages from misconfigured changes. Integrate with OneUptime to monitor post-deployment health and page on-call engineers if checks fail after deployment.
