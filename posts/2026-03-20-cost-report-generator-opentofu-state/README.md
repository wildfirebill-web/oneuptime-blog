# How to Build a Cost Report Generator from OpenTofu State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Cost Management, FinOps, Reporting, Python, Infrastructure as Code

Description: Learn how to generate cost attribution reports by parsing OpenTofu state files and correlating resources with AWS Cost Explorer data.

## Introduction

OpenTofu state contains a complete inventory of your managed resources with their tags. By combining state data with AWS Cost Explorer, you can generate per-team, per-environment, and per-project cost reports without manual tagging audits.

## Extracting Resources from State

```bash
#!/bin/bash
# Step 1: Export state as JSON for processing

tofu state pull > /tmp/terraform.tfstate.json

# Or for all workspaces

for workspace in $(tofu workspace list | tr -d '* '); do
  tofu workspace select "$workspace"
  tofu state pull > "/tmp/state-${workspace}.json"
done
```

## Python Cost Report Generator

```python
#!/usr/bin/env python3
# scripts/cost_report.py

import json
import boto3
from datetime import datetime, timedelta
from collections import defaultdict

def load_state(state_file: str) -> dict:
    """Load and parse OpenTofu state JSON."""
    with open(state_file) as f:
        return json.load(f)

def extract_resources(state: dict) -> list[dict]:
    """Extract resource metadata from state."""
    resources = []

    for resource in state.get("resources", []):
        # Skip data sources and meta resources
        if resource.get("mode") != "managed":
            continue

        for instance in resource.get("instances", []):
            attrs = instance.get("attributes", {})
            tags = attrs.get("tags", {}) or attrs.get("tags_all", {}) or {}

            resources.append({
                "type": resource["type"],
                "name": resource["name"],
                "module": resource.get("module", "root"),
                "id": attrs.get("id", ""),
                "arn": attrs.get("arn", ""),
                "environment": tags.get("Environment", "unknown"),
                "team": tags.get("Team", "unknown"),
                "cost_center": tags.get("CostCenter", "unknown"),
                "project": tags.get("Project", "unknown"),
            })

    return resources

def get_cost_by_tags(cost_client, start: str, end: str, tag_key: str) -> dict:
    """Retrieve cost breakdown by a specific tag using Cost Explorer."""
    response = cost_client.get_cost_and_usage(
        TimePeriod={"Start": start, "End": end},
        Granularity="MONTHLY",
        Metrics=["UnblendedCost"],
        GroupBy=[{"Type": "TAG", "Key": tag_key}]
    )

    costs = {}
    for result in response.get("ResultsByTime", []):
        for group in result.get("Groups", []):
            tag_value = group["Keys"][0].replace(f"{tag_key}$", "")
            amount = float(group["Metrics"]["UnblendedCost"]["Amount"])
            currency = group["Metrics"]["UnblendedCost"]["Unit"]
            costs[tag_value or "untagged"] = {"amount": amount, "currency": currency}

    return costs

def generate_report(state_file: str, output_file: str):
    """Generate a cost attribution report."""
    state = load_state(state_file)
    resources = extract_resources(state)

    # Summary by environment
    env_summary = defaultdict(int)
    for r in resources:
        env_summary[r["environment"]] += 1

    # Get actual costs from AWS Cost Explorer
    cost_client = boto3.client("ce", region_name="us-east-1")

    end_date = datetime.now().strftime("%Y-%m-%d")
    start_date = (datetime.now() - timedelta(days=30)).strftime("%Y-%m-%d")

    team_costs = get_cost_by_tags(cost_client, start_date, end_date, "Team")
    env_costs = get_cost_by_tags(cost_client, start_date, end_date, "Environment")

    # Build report
    report = {
        "generated_at": datetime.now().isoformat(),
        "period": {"start": start_date, "end": end_date},
        "resource_count": len(resources),
        "resources_by_environment": dict(env_summary),
        "cost_by_environment": env_costs,
        "cost_by_team": team_costs,
    }

    with open(output_file, "w") as f:
        json.dump(report, f, indent=2)

    # Print summary
    print(f"\n=== Cost Report ({start_date} to {end_date}) ===")
    print(f"Total resources: {len(resources)}")
    print("\nCost by Team:")
    for team, data in sorted(team_costs.items(), key=lambda x: -x[1]["amount"]):
        print(f"  {team:<30} ${data['amount']:>10.2f} {data['currency']}")

if __name__ == "__main__":
    import sys
    state_file = sys.argv[1] if len(sys.argv) > 1 else "/tmp/terraform.tfstate.json"
    output_file = sys.argv[2] if len(sys.argv) > 2 else "/tmp/cost_report.json"
    generate_report(state_file, output_file)
    print(f"\nReport saved to: {output_file}")
```

## Running the Report

```bash
# Pull state and generate report
tofu state pull > /tmp/terraform.tfstate.json
python3 scripts/cost_report.py /tmp/terraform.tfstate.json /tmp/cost_report.json

# Schedule weekly via cron
# 0 9 * * 1 /path/to/run-cost-report.sh >> /var/log/cost-report.log
```

## Summary

Combining OpenTofu state data with AWS Cost Explorer provides accurate cost attribution by team, environment, and project. The state file serves as a ground truth resource inventory, while Cost Explorer provides the actual spend data - together enabling FinOps-grade reporting without additional tooling.
