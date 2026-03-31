# How to Avoid Unnecessary Data Source Calls in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Data Source, Performance, Best Practice, Infrastructure as Code

Description: Learn how to avoid unnecessary data source calls in OpenTofu that slow down plans, hit API rate limits, and add complexity without benefit.

## Introduction

Data sources in OpenTofu read information from external systems during planning. While useful, unnecessary data source calls make plans slower, consume API rate limits, add complexity, and can cause plan failures when APIs are unavailable. Knowing when to use a data source versus a variable or local value is essential for efficient configurations.

## When Data Sources Are Appropriate

```text
Use data sources when:
✓ Looking up resources created outside your configuration
✓ Getting dynamically changing values (latest AMI, current region)
✓ Reading values that exist in cloud APIs but not in your config
✓ Referencing shared infrastructure from another state

Don't use data sources when:
✗ You already know the value and it rarely changes
✗ The value is a simple derivation of a variable
✗ You're adding a data source just to "document" a value
✗ You have the resource in the same state file
```

## Common Unnecessary Data Source: AMI Lookup Every Plan

Looking up AMIs on every plan adds latency and API calls.

```hcl
# UNNECESSARY: AMI rarely changes, repeated API call

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-22.04-amd64-server-*"]
  }
}

resource "aws_launch_template" "app" {
  image_id = data.aws_ami.ubuntu.id
}

# BETTER: Pin the AMI ID as a variable, update intentionally
variable "base_ami_id" {
  type        = string
  description = "Ubuntu 22.04 LTS AMI ID for the target region"
  default     = "ami-0c7217cdde317cfec"  # update when OS patches are needed
}

resource "aws_launch_template" "app" {
  image_id = var.base_ami_id
}
```

## Common Unnecessary Data Source: Self-Referencing Resources

Reading a resource from a data source when it's in the same configuration.

```hcl
# UNNECESSARY: You already have a reference to this resource
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

data "aws_vpc" "main" {
  id = aws_vpc.main.id  # redundant - you already have the resource!
}

resource "aws_subnet" "public" {
  vpc_id = data.aws_vpc.main.id  # unnecessary indirection
}

# CORRECT: Reference the resource directly
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id  # direct reference, no API call needed
}
```

## Common Unnecessary Data Source: Region and Account Lookups

Only call these once and cache in a local.

```hcl
# WASTEFUL: Multiple data source calls for the same values
data "aws_region" "current" {}
data "aws_caller_identity" "current1" {}  # called in module A
data "aws_caller_identity" "current2" {}  # called in module B (duplicate!)

# BETTER: Call once per configuration, pass via variables
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name
}

# Pass to modules as variables rather than having each module call its own
module "storage" {
  source     = "./modules/storage"
  account_id = local.account_id
  region     = local.region
}
```

## Caching Slow Data Sources with -refresh=false

Skip data source refresh when you know nothing has changed.

```bash
# Skip refreshing data sources (use cached state values)
tofu plan -refresh=false

# This is useful when:
# - Working offline or with limited network
# - Plans are very slow due to many data source API calls
# - You're debugging configuration logic, not infrastructure state
```

## Data Sources That Scan Many Resources

Some data sources make expensive scans - avoid calling them in frequently-run configs.

```hcl
# EXPENSIVE: Scans all subnets every plan
data "aws_subnets" "all_private" {
  filter {
    name   = "tag:Type"
    values = ["private"]
  }
}

# BETTER: Use variables to pass subnet IDs explicitly
variable "private_subnet_ids" {
  type        = list(string)
  description = "List of private subnet IDs"
}
```

## Summary

Unnecessary data source calls slow plans, hit API rate limits, and add complexity. Avoid data sources for values you already have in your configuration, values that rarely change (pin them as variables instead), and resources that are in the same state file. When data sources are necessary, call them once and pass values through locals or module variables rather than duplicating the same data source across multiple files or modules. Use `-refresh=false` during development to skip data source refreshes and speed up planning.
