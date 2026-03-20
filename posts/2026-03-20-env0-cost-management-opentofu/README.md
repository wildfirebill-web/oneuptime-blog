# How to Use env0 for Cost Management with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, env0, Cost Management, FinOps, Infrastructure as Code, DevOps

Description: Learn how to configure env0 to run OpenTofu deployments and leverage its built-in cost estimation and budget enforcement features to keep cloud spend under control.

## Introduction

env0 is a self-service infrastructure automation platform that natively supports OpenTofu. Its cost management features surface Infracost estimates on every pull request and let platform teams set hard budget limits that block deployments exceeding defined thresholds.

## Connecting Your Repository to env0

Create an env0 environment via its OpenTofu provider:

```hcl
# env0.tf
terraform {
  required_providers {
    env0 = {
      source  = "env0/env0"
      version = "~> 1.0"
    }
  }
}

provider "env0" {
  # API key set via ENV0_API_KEY environment variable
}

# Create a project
resource "env0_project" "platform" {
  name = "platform-infrastructure"
}

# Create an environment that uses OpenTofu
resource "env0_environment" "production" {
  name           = "production"
  project_id     = env0_project.platform.id
  template_id    = env0_template.vpc.id
  auto_deploy    = false
  # Use OpenTofu runtime
  opentofu_version = "1.9.0"
}
```

## Registering an OpenTofu Template

```hcl
resource "env0_template" "vpc" {
  name       = "aws-vpc"
  type       = "opentofu"
  repository = "https://github.com/my-org/infra-repo"
  path       = "modules/vpc"
  branch     = "main"

  # Override the runner to use OpenTofu
  terraform_version = ""  # Leave blank when using opentofu_version
  opentofu_version  = "1.9.0"
}
```

## Setting Up Cost Estimation

env0 integrates Infracost under the hood. Enable it in your environment settings:

```hcl
resource "env0_environment" "production" {
  name       = "production"
  project_id = env0_project.platform.id
  template_id = env0_template.vpc.id

  # Enable cost estimation on every plan
  cost_estimation_enabled = true
}
```

Each pull request plan will now include a cost breakdown comment like:

```
Monthly cost estimate: $142.50
  aws_instance.web (t3.medium): $30.22/month
  aws_db_instance.main (db.t3.small): $27.60/month
  aws_nat_gateway.main: $32.40/month
  ...
```

## Budget Policies

Set a monthly budget limit that blocks deployments over threshold:

```hcl
resource "env0_cost_credentials" "aws" {
  name    = "aws-cost-creds"
  type    = "AWS_ASSUMED_ROLE_FOR_BILLING"
  project_id = env0_project.platform.id

  aws_assumed_role_arn = "arn:aws:iam::123456789012:role/env0-billing-role"
}

resource "env0_environment_discovery_configuration" "prod" {
  project_id          = env0_project.platform.id
  max_ttl             = "24-h"          # Auto-destroy after 24 hours for ephemeral envs
  default_ttl         = "8-h"
}
```

## Variable Management for Cost Optimization

Use env0 variable sets to enforce cost-saving instance types by environment:

```hcl
resource "env0_variable_set" "dev_cost_controls" {
  name       = "dev-cost-controls"
  project_id = env0_project.platform.id

  variable {
    name  = "instance_type"
    value = "t3.micro"
  }
  variable {
    name  = "rds_instance_class"
    value = "db.t3.micro"
  }
}

resource "env0_variable_set" "prod_cost_controls" {
  name       = "prod-cost-controls"
  project_id = env0_project.platform.id

  variable {
    name  = "instance_type"
    value = "m5.large"
  }
  variable {
    name  = "rds_instance_class"
    value = "db.r5.large"
  }
}
```

## TTL-Based Auto-Destroy for Ephemeral Environments

Auto-destroying short-lived environments is one of the most effective cost controls:

```hcl
resource "env0_environment" "feature_branch" {
  name        = "feature-review"
  project_id  = env0_project.platform.id
  template_id = env0_template.vpc.id

  # Automatically destroy after 8 hours
  ttl_request {
    type  = "HOURS"
    value = "8"
  }
}
```

## Conclusion

env0 makes cost management a first-class concern in your OpenTofu workflow. Cost estimates surface on every PR, budget policies block runaway spending before deployment, and TTL-based auto-destroy keeps ephemeral environments from accumulating idle cloud costs.
