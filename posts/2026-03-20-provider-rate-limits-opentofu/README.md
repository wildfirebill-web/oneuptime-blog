# How to Handle Provider Rate Limits in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Rate Limit, Provider, AWS, GitHub, Best Practice

Description: Learn how to handle provider-specific rate limits in OpenTofu for AWS, GitHub, and other providers using retry configuration and request batching strategies.

## Introduction

Every cloud provider and third-party API has rate limits. OpenTofu providers implement retry logic, but large applies can still exhaust limits. This guide covers provider-specific rate limit handling and general mitigation strategies.

## AWS Provider Rate Limits

```hcl
provider "aws" {
  region = var.region

  # Increase retry attempts (default: 25)
  max_retries = 30

  # Use environment variables for credentials to avoid extra API calls
  # AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_PROFILE
}
```

Common AWS rate-limited services:
- **IAM**: 10 req/sec for write operations
- **EC2**: 100 req/sec for most describe operations
- **CloudWatch**: 10 req/sec for PutMetricAlarm

```bash
# For IAM-heavy applies, reduce parallelism

tofu apply -parallelism=3 -var-file=production.tfvars
```

## GitHub Provider Rate Limits

```hcl
provider "github" {
  token = var.github_token
  owner = var.github_org

  # GitHub API rate limit: 5,000 req/hour for authenticated calls
  # For large org configurations (many repos/teams), use a GitHub App token
  # which has higher limits
}
```

For organizations with many repositories:

```hcl
# Instead of creating all repos in one apply, use targeted applies
# or batch them by team
module "team_alpha_repos" {
  source = "./modules/github-repos"
  repos  = var.team_alpha_repos
}

# Apply separately: tofu apply -target=module.team_alpha_repos
```

## Datadog Provider Rate Limits

```hcl
provider "datadog" {
  api_key = var.datadog_api_key
  app_key = var.datadog_app_key

  # Datadog has ~3,000 API requests/hour
  # For large monitor configurations, batch applies
}

# Create monitors in batches using for_each with deliberate ordering
resource "datadog_monitor" "service_monitors" {
  for_each = var.service_monitors
  # ...
}
```

## General Rate Limit Mitigation Patterns

### 1. Request Batching

Where possible, use resource types that batch multiple operations.

```hcl
# Instead of individual IAM policy attachments (one API call each)
resource "aws_iam_role_policy_attachment" "individual" {
  count      = length(var.policy_arns)
  role       = aws_iam_role.app.name
  policy_arn = var.policy_arns[count.index]
}

# Use inline policy (single API call) for simpler cases
resource "aws_iam_role_policy" "inline" {
  role   = aws_iam_role.app.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [for arn in var.policy_statements : arn]
  })
}
```

### 2. Apply During Off-Peak Hours

```bash
# Schedule large applies during low-traffic hours
# Use at command (Linux/macOS)
echo "tofu apply -auto-approve production.tfplan" | at 02:00
```

### 3. Caching with Data Sources

Avoid repeated API calls for the same data.

```hcl
# Cache AMI lookups using locals
locals {
  # This data source is only called once and cached
  ubuntu_ami_id = data.aws_ami.ubuntu.id
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-22.04-amd64-server-*"]
  }
}

# All instances use the cached value
resource "aws_instance" "servers" {
  count = 20
  ami   = local.ubuntu_ami_id  # single API call, cached for all 20 instances
  # ...
}
```

## Detecting Rate Limit Errors

```bash
# Enable debug logging to see rate limit errors
TF_LOG=DEBUG tofu apply 2>&1 | grep -i "throttl\|rate limit\|429\|RequestThrottled"
```

## Summary

Provider rate limits require a combination of reduced parallelism, increased retry counts, request batching, and off-peak scheduling for large applies. Understanding each provider's specific limits and using cached data sources reduces unnecessary API calls - making large applies more reliable.
