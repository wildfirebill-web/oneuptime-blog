# How to Group Resources by Attribute Using for Expressions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, For Expressions, Collections, Grouping, Infrastructure

Description: Learn how to group resources and data by attribute values using for expressions in OpenTofu, enabling environment-based organization, multi-key aggregation, and data partitioning.

## Overview

Grouping resources by a shared attribute - such as environment, region, or team - is a common pattern when managing infrastructure at scale. OpenTofu's `for` expressions support grouping with the `...` (ellipsis) operator, which aggregates multiple values with the same key into a list instead of overwriting them.

## Basic Grouping with the Ellipsis Operator

```hcl
# Group a list of resources by a shared attribute

locals {
  all_services = [
    { name = "api",       env = "prod",    team = "platform" },
    { name = "worker",    env = "prod",    team = "backend" },
    { name = "scheduler", env = "staging", team = "backend" },
    { name = "frontend",  env = "prod",    team = "frontend" },
    { name = "api",       env = "staging", team = "platform" },
  ]

  # Group services by environment
  # Result: { "prod" = [...], "staging" = [...] }
  services_by_env = {
    for svc in local.all_services :
    svc.env => svc...  # The ... groups multiple values under the same key
  }
}

output "prod_services" {
  value = local.services_by_env["prod"]
  # [
  #   { name = "api",      env = "prod", team = "platform" },
  #   { name = "worker",   env = "prod", team = "backend" },
  #   { name = "frontend", env = "prod", team = "frontend" },
  # ]
}

output "environments" {
  value = keys(local.services_by_env)  # ["prod", "staging"]
}
```

## Group by Environment for Per-Environment Resources

```hcl
variable "service_configs" {
  type = list(object({
    name     = string
    env      = string
    memory   = number
    replicas = number
  }))
  default = [
    { name = "api",    env = "prod",    memory = 1024, replicas = 3 },
    { name = "worker", env = "prod",    memory = 512,  replicas = 2 },
    { name = "api",    env = "staging", memory = 512,  replicas = 1 },
    { name = "worker", env = "staging", memory = 256,  replicas = 1 },
  ]
}

locals {
  # Group configs by environment
  configs_by_env = {
    for cfg in var.service_configs :
    cfg.env => cfg...
  }

  # Compute per-environment resource totals
  env_resource_totals = {
    for env, services in local.configs_by_env :
    env => {
      total_memory   = sum([for svc in services : svc.memory * svc.replicas])
      total_replicas = sum([for svc in services : svc.replicas])
      service_count  = length(services)
    }
  }
}

output "environment_resource_summary" {
  value = local.env_resource_totals
  # {
  #   prod    = { total_memory = 4096, total_replicas = 5, service_count = 2 }
  #   staging = { total_memory = 768,  total_replicas = 2, service_count = 2 }
  # }
}
```

## Group Subnets by Availability Zone

```hcl
data "aws_subnets" "all" {
  filter {
    name   = "vpc-id"
    values = [var.vpc_id]
  }
}

data "aws_subnet" "details" {
  for_each = toset(data.aws_subnets.all.ids)
  id       = each.value
}

locals {
  # Group subnet IDs by AZ
  subnets_by_az = {
    for id, subnet in data.aws_subnet.details :
    subnet.availability_zone => id...
  }

  # Get one subnet per AZ for node group placement
  one_subnet_per_az = {
    for az, subnet_ids in local.subnets_by_az :
    az => subnet_ids[0]
  }
}

resource "aws_eks_node_group" "per_az" {
  for_each = local.one_subnet_per_az

  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "ng-${each.key}"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = [each.value]

  scaling_config {
    desired_size = 2
    min_size     = 1
    max_size     = 5
  }
}
```

## Group IAM Policies by Team

```hcl
locals {
  team_permissions = [
    { team = "platform",  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess" },
    { team = "platform",  policy_arn = "arn:aws:iam::aws:policy/AWSBillingReadOnlyAccess" },
    { team = "backend",   policy_arn = "arn:aws:iam::aws:policy/AmazonECS_FullAccess" },
    { team = "backend",   policy_arn = "arn:aws:iam::aws:policy/AmazonRDSFullAccess" },
    { team = "frontend",  policy_arn = "arn:aws:iam::aws:policy/CloudFrontFullAccess" },
    { team = "data",      policy_arn = "arn:aws:iam::aws:policy/AmazonAthenaFullAccess" },
    { team = "data",      policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess" },
  ]

  # Group policy ARNs by team
  policies_by_team = {
    for item in local.team_permissions :
    item.team => item.policy_arn...
  }
}

# Attach all policies to each team's role
resource "aws_iam_role_policy_attachment" "team_policies" {
  for_each = {
    for combo in flatten([
      for team, arns in local.policies_by_team : [
        for arn in arns : {
          key  = "${team}/${basename(arn)}"
          team = team
          arn  = arn
        }
      ]
    ]) : combo.key => combo
  }

  role       = aws_iam_role.team_roles[each.value.team].name
  policy_arn = each.value.arn
}
```

## Extract Names from Grouped Data

```hcl
locals {
  instances = [
    { name = "web-1", type = "web",      az = "us-east-1a" },
    { name = "web-2", type = "web",      az = "us-east-1b" },
    { name = "db-1",  type = "database", az = "us-east-1a" },
    { name = "cache", type = "cache",    az = "us-east-1b" },
  ]

  # Group instance names by type
  names_by_type = {
    for inst in local.instances :
    inst.type => inst.name...
  }
}

output "web_instance_names" {
  value = local.names_by_type["web"]  # ["web-1", "web-2"]
}

output "instance_counts_by_type" {
  value = {
    for type, names in local.names_by_type :
    type => length(names)
  }
  # { web = 2, database = 1, cache = 1 }
}
```

## Summary

The `for` expression with the `...` grouping operator is the idiomatic OpenTofu way to partition flat lists into categorized maps. Common use cases include grouping services by environment for capacity planning, organizing subnets by AZ for placement decisions, and aggregating IAM permissions by team. Once grouped, you can apply `sum`, `length`, and further `for` expressions to derive computed metadata from each partition.
