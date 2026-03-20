# How to Flatten Nested Data Structures for for_each in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Flatten, for_each, Collections, Functions

Description: Learn how to use the flatten function and for expressions in OpenTofu to convert nested data structures into flat maps suitable for for_each-based resource creation.

## Overview

OpenTofu's `for_each` requires a flat map or set. When configuration data is naturally hierarchical - like environments containing services, regions containing subnets, or teams containing members - you must flatten it before iterating. The `flatten` function combined with `for` expressions is the idiomatic solution.

## Basic flatten Usage

```hcl
# Without flatten: nested lists

locals {
  nested = [
    ["a", "b", "c"],
    ["d", "e"],
    ["f"],
  ]

  flat = flatten(local.nested)
  # ["a", "b", "c", "d", "e", "f"]
}

# With for expressions inside: generates a nested list automatically
locals {
  environments = ["prod", "staging", "dev"]
  az_suffixes  = ["a", "b", "c"]

  # flatten([...]) unwraps the outer list created by the outer for
  all_combinations = flatten([
    for env in local.environments : [
      for az in local.az_suffixes : "${env}-${az}"
    ]
  ])
  # ["prod-a", "prod-b", "prod-c", "staging-a", "staging-b", ...]
}
```

## Flatten Environments with Services into for_each Map

```hcl
variable "environment_services" {
  type = map(list(object({
    name     = string
    replicas = number
    port     = number
  })))
  default = {
    prod = [
      { name = "api",    replicas = 3, port = 8080 },
      { name = "worker", replicas = 2, port = 0 },
    ]
    staging = [
      { name = "api",    replicas = 1, port = 8080 },
      { name = "worker", replicas = 1, port = 0 },
    ]
  }
}

locals {
  # Flatten into a list of objects with env embedded
  flat_services = flatten([
    for env, services in var.environment_services : [
      for svc in services : {
        key      = "${env}/${svc.name}"
        env      = env
        name     = svc.name
        replicas = svc.replicas
        port     = svc.port
      }
    ]
  ])

  # Convert to map keyed by composite key for for_each
  services_map = {
    for item in local.flat_services :
    item.key => item
  }
}

resource "kubernetes_deployment" "services" {
  for_each = local.services_map

  metadata {
    name      = "${each.value.name}-${each.value.env}"
    namespace  = each.value.env
    labels = {
      app = each.value.name
      env = each.value.env
    }
  }

  spec {
    replicas = each.value.replicas

    selector {
      match_labels = {
        app = each.value.name
      }
    }

    template {
      metadata {
        labels = { app = each.value.name }
      }
      spec {
        container {
          name  = each.value.name
          image = "myregistry/${each.value.name}:latest"
        }
      }
    }
  }
}
```

## Flatten Team Members for IAM Resources

```hcl
locals {
  team_members = {
    platform = {
      role    = "PowerUserAccess"
      members = ["alice", "bob"]
    }
    data = {
      role    = "ReadOnlyAccess"
      members = ["carol", "dave", "eve"]
    }
    security = {
      role    = "SecurityAudit"
      members = ["frank"]
    }
  }

  # Flatten to one object per team+member combination
  flat_memberships = flatten([
    for team, config in local.team_members : [
      for member in config.members : {
        key    = "${team}/${member}"
        team   = team
        member = member
        role   = config.role
      }
    ]
  ])

  memberships_map = {
    for item in local.flat_memberships :
    item.key => item
  }
}

resource "aws_iam_user_group_membership" "memberships" {
  for_each = local.memberships_map

  user = aws_iam_user.users[each.value.member].name
  groups = [aws_iam_group.groups[each.value.team].name]
}
```

## Flatten Security Group Rules from Multiple Services

```hcl
variable "services" {
  type = list(object({
    name         = string
    allowed_ports = list(number)
    allowed_cidrs = list(string)
  }))
  default = [
    {
      name          = "web"
      allowed_ports = [80, 443]
      allowed_cidrs = ["0.0.0.0/0"]
    },
    {
      name          = "api"
      allowed_ports = [8080, 8443]
      allowed_cidrs = ["10.0.0.0/8"]
    },
  ]
}

locals {
  # Generate one entry per service+port+cidr combination
  sg_rules = flatten([
    for svc in var.services : [
      for port in svc.allowed_ports : [
        for cidr in svc.allowed_cidrs : {
          key  = "${svc.name}-${port}-${replace(cidr, "/", "_")}"
          name = svc.name
          port = port
          cidr = cidr
        }
      ]
    ]
  ])

  sg_rules_map = {
    for rule in local.sg_rules :
    rule.key => rule
  }
}

resource "aws_security_group_rule" "ingress" {
  for_each = local.sg_rules_map

  type              = "ingress"
  from_port         = each.value.port
  to_port           = each.value.port
  protocol          = "tcp"
  cidr_blocks       = [each.value.cidr]
  security_group_id = aws_security_group.services[each.value.name].id
}
```

## Flatten DNS Records from Multiple Zones

```hcl
locals {
  dns_zones = {
    "example.com" = {
      records = [
        { type = "A",   name = "www",  value = "1.2.3.4",            ttl = 300 },
        { type = "A",   name = "api",  value = "1.2.3.5",            ttl = 300 },
        { type = "MX",  name = "@",    value = "10 mail.example.com", ttl = 3600 },
      ]
    }
    "example.net" = {
      records = [
        { type = "A",     name = "www",  value = "1.2.3.4", ttl = 300 },
        { type = "CNAME", name = "docs", value = "docs.example.com", ttl = 600 },
      ]
    }
  }

  flat_dns_records = flatten([
    for zone, config in local.dns_zones : [
      for record in config.records : {
        key   = "${zone}/${record.type}/${record.name}"
        zone  = zone
        type  = record.type
        name  = record.name
        value = record.value
        ttl   = record.ttl
      }
    ]
  ])

  dns_records_map = {
    for rec in local.flat_dns_records :
    rec.key => rec
  }
}

resource "aws_route53_record" "records" {
  for_each = local.dns_records_map

  zone_id = aws_route53_zone.zones[each.value.zone].zone_id
  name    = "${each.value.name}.${each.value.zone}"
  type    = each.value.type
  ttl     = each.value.ttl
  records = [each.value.value]
}
```

## Summary

The `flatten` function combined with nested `for` expressions is the standard OpenTofu pattern for generating `for_each`-compatible maps from hierarchical data. The three-step workflow - build nested list with `flatten([for outer : [for inner : {...}]])`, then convert to a map with a unique composite key - handles any depth of nesting. This pattern is especially valuable for generating security group rules, DNS records, IAM bindings, and Kubernetes resources where the source data naturally groups by multiple dimensions.
