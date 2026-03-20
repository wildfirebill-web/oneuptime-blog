# How to Use OpenTofu with env0 Cost Controls

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, env0, Cost Management, FinOps, Cloud Cost Controls

Description: Learn how to configure env0's cost estimation and control features with OpenTofu to enforce budget limits, require approvals for expensive deployments, and track infrastructure costs.

## Introduction

env0 integrates with OpenTofu to provide cost estimation at plan time. You can set budget thresholds that trigger warnings, require manual approval for deployments that exceed cost limits, and track actual vs estimated costs over time.

## Configuring Cost Estimation in env0

```yaml
# .env0/configuration.yml
version: 2

cost_estimation:
  enabled: true

environments:
  - name: production
    opentofu_version: "1.7.0"
    workspace: prod

    # Cost thresholds
    cost_estimation:
      monthly_budget: 5000  # USD

      # Auto-approve if monthly cost change is below $100
      auto_approve_threshold: 100

      # Require approval if cost increases by more than 10%
      approval_required_threshold_percentage: 10
```

## Cost Policy via env0 API

```hcl
# Configure env0 project with cost controls via Terraform provider
provider "env0" {
  api_key    = var.env0_api_key
  api_secret = var.env0_api_secret
}

resource "env0_project" "production" {
  name        = "Production Infrastructure"
  description = "Production OpenTofu configurations"
}

resource "env0_cost_credentials" "aws" {
  name       = "AWS Cost Credentials"
  project_id = env0_project.production.id
}
```

## Environment TTL for Cost Savings

env0 supports automatic environment destruction to save costs in non-production environments:

```yaml
# .env0/configuration.yml
environments:
  - name: feature-branch
    opentofu_version: "1.7.0"

    # Auto-destroy feature branch environments after 8 hours
    ttl:
      type: hours
      value: 8

  - name: staging
    ttl:
      type: business_days
      value: 5  # Keep for 5 business days

  - name: production
    # No TTL for production
```

## Cost Notification Configuration

```yaml
# Notify when cost estimates change significantly
notifications:
  - type: slack
    channel: "#infrastructure-costs"
    events:
      - cost_exceeds_budget
      - cost_increased_by_percentage:
          threshold: 15
```

## Tagging for Cost Attribution

Ensure resources have cost tags that map to env0 environments:

```hcl
# Add env0 environment context to tags
locals {
  env0_tags = {
    "env0:environment"  = var.env0_environment_name
    "env0:project"      = var.env0_project_name
    "env0:workspace"    = terraform.workspace
  }
}

resource "aws_instance" "app" {
  # ...
  tags = merge(local.env0_tags, var.application_tags)
}
```

## Cost Report Module

```hcl
# Use env0 API to get cost reports programmatically
data "http" "cost_report" {
  url    = "https://api.env0.com/environments/${var.env0_environment_id}/cost"
  method = "GET"

  request_headers = {
    Authorization = "Bearer ${var.env0_api_token}"
  }
}

output "current_monthly_cost" {
  value = jsondecode(data.http.cost_report.response_body).monthlyToDate
}
```

## Conclusion

env0 cost controls integrate seamlessly with OpenTofu by analyzing plan output before apply. The combination of automatic cost estimation, configurable approval thresholds, and environment TTLs creates a complete FinOps workflow for infrastructure teams. Start with monitoring-only mode to establish cost baselines before enabling approval gates.
