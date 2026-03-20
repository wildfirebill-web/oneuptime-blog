# How to Manage Large State Files for Performance in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, State

Description: Learn how to manage large OpenTofu state files for better performance using state splitting, targeted operations, and architectural patterns to keep state manageable.

## Introduction

As infrastructure grows, state files can become large - sometimes containing thousands of resources. Large state files slow down every OpenTofu operation because each plan must process the entire state. This guide covers strategies to keep state files manageable and operations fast.

## Measuring State Size

```bash
# Count resources in state

tofu state list | wc -l

# Measure state file size
tofu state pull | wc -c

# Find the largest resources by attribute count
tofu state pull | jq '
  .resources[]
  | {type: .type, name: .name, attrs: (.instances[0].attributes | length)}
  | select(.attrs > 20)
'
```

## Strategy 1: Split State by Layer

The most effective solution is architectural - split one large state into multiple smaller states:

```text
Before: One state file with 500 resources
After:
  networking/terraform.tfstate  (50 resources)
  eks/terraform.tfstate         (100 resources)
  apps/terraform.tfstate        (200 resources)
  monitoring/terraform.tfstate  (50 resources)
```

```hcl
# networking/ - manages VPC, subnets, security groups
terraform {
  backend "s3" {
    bucket = "my-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}
```

```hcl
# eks/ - reads networking outputs, manages cluster
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_eks_cluster" "main" {
  vpc_config {
    subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
  }
}
```

## Strategy 2: Use -target for Focused Operations

Target specific resources to avoid loading the full dependency graph:

```bash
# Plan only for a specific resource
tofu plan -target=module.new_feature

# Apply only a specific resource
tofu apply -target=aws_s3_bucket.new_bucket
```

Note: `-target` is for exceptional cases; splitting state is the proper solution.

## Strategy 3: Parallelism Tuning

```bash
# Reduce parallelism to decrease memory usage (default is 10)
tofu plan -parallelism=5

# Increase for faster plans when resources don't conflict
tofu plan -parallelism=20
```

## Strategy 4: Skip Refresh for Known-Good State

```bash
# Skip the refresh step to speed up planning when state is trusted
tofu plan -refresh=false

# Only use when you are confident state is accurate
```

## Strategy 5: Use for_each Over count

`for_each` groups related resources, making plan output clearer for large resource sets:

```hcl
# Better performance characteristics than count for many instances
resource "aws_iam_user" "developers" {
  for_each = toset(var.developer_emails)
  name     = each.key
}
```

## Identifying Resources to Extract

```bash
# Find the modules with the most resources
tofu state list | sed 's/\.[^.]*$//' | sort | uniq -c | sort -rn | head -20
```

## State Splitting Migration

```bash
# Step 1: Move resources to the new state file
tofu state mv \
  -state-out=networking/terraform.tfstate \
  module.vpc.aws_vpc.this \
  aws_vpc.main

# Step 2: Update the configuration to split correctly
# Step 3: Import resources if needed
# Step 4: Verify with tofu plan in each state
```

## Conclusion

Large state files are a performance and operational risk. The most impactful solution is splitting state by architectural layer - networking, compute, data, and monitoring each in their own state file. Use `terraform_remote_state` data sources to pass outputs between layers. This pattern scales to hundreds of engineers working on the same infrastructure without performance degradation.
