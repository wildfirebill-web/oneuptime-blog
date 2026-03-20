# How to Use Nested Loops with for Expressions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, for expressions, Nested Loops, HCL, Data Transformation

Description: Learn how to use nested for expressions in OpenTofu to create cartesian products and flatten complex nested data structures into resource-ready formats.

## Introduction

Real infrastructure often requires creating resources for every combination of two lists — every region paired with every environment, every user paired with every role, etc. Nested `for` expressions let you compute these cartesian products directly in HCL.

## Cartesian Product with Nested for

Create all combinations of two lists using a nested `for` expression.

```hcl
variable "environments" {
  type    = list(string)
  default = ["dev", "staging", "prod"]
}

variable "regions" {
  type    = list(string)
  default = ["us-east-1", "eu-west-1"]
}

locals {
  # Create all region-environment combinations as a flat map
  env_region_pairs = {
    for pair in flatten([
      for env in var.environments : [
        for region in var.regions : {
          key    = "${env}-${region}"
          env    = env
          region = region
        }
      ]
    ]) : pair.key => pair
  }
}

# Create an S3 bucket for every env-region combination
resource "aws_s3_bucket" "regional_env" {
  for_each = local.env_region_pairs

  provider = aws.regional[each.value.region]
  bucket   = "myapp-${each.value.env}-${each.value.region}-artifacts"

  tags = {
    Environment = each.value.env
    Region      = each.value.region
  }
}
```

## User-Role Assignment Matrix

```hcl
variable "users" {
  type = list(object({
    name    = string
    team    = string
  }))
  default = [
    { name = "alice", team = "backend"  },
    { name = "bob",   team = "frontend" },
  ]
}

variable "team_roles" {
  type = map(list(string))
  default = {
    "backend"  = ["arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"]
    "frontend" = ["arn:aws:iam::aws:policy/CloudFrontReadOnlyAccess"]
  }
}

locals {
  # Create a flat list of user-policy pairs
  user_policy_attachments = flatten([
    for user in var.users : [
      for policy_arn in lookup(var.team_roles, user.team, []) : {
        key        = "${user.name}-${basename(policy_arn)}"
        user_name  = user.name
        policy_arn = policy_arn
      }
    ]
  ])
}

resource "aws_iam_user_policy_attachment" "team_policies" {
  for_each = {
    for item in local.user_policy_attachments : item.key => item
  }

  user       = aws_iam_user.users[each.value.user_name].name
  policy_arn = each.value.policy_arn
}
```

## Flattening Nested Structures

```hcl
variable "vpc_subnets" {
  type = map(object({
    cidr_ranges = list(string)
    tier        = string
  }))
  default = {
    "us-east-1a" = { cidr_ranges = ["10.0.1.0/24", "10.0.2.0/24"], tier = "private" }
    "us-east-1b" = { cidr_ranges = ["10.0.3.0/24", "10.0.4.0/24"], tier = "private" }
  }
}

locals {
  # Flatten to one entry per CIDR block
  all_subnet_configs = {
    for item in flatten([
      for az, config in var.vpc_subnets : [
        for idx, cidr in config.cidr_ranges : {
          key  = "${az}-${idx}"
          az   = az
          cidr = cidr
          tier = config.tier
        }
      ]
    ]) : item.key => item
  }
}

resource "aws_subnet" "all" {
  for_each = local.all_subnet_configs

  vpc_id            = var.vpc_id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  tags = {
    Name = "subnet-${each.key}"
    Tier = each.value.tier
  }
}
```

## Conclusion

Nested `for` expressions combined with `flatten` and map construction let you compute complex data transformations entirely in HCL. The key pattern is: nested loops produce a list of lists, `flatten` collapses it to a flat list, and a final `for` expression converts it to a map keyed by a unique composite key. This three-step pattern handles most cartesian product and data reshaping needs.
