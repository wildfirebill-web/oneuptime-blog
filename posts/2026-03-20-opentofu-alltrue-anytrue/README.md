---
title: "How to Use alltrue() and anytrue() in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, functions
description: "Learn how to use the alltrue() and anytrue() functions in OpenTofu to evaluate boolean conditions across a collection."
---

# How to Use alltrue() and anytrue() in OpenTofu

`alltrue()` returns true if all elements in a list are true, and `anytrue()` returns true if at least one element is true. They're commonly used to validate conditions across collections.

## alltrue()

Returns `true` only if every element in the list is `true`:

```hcl
> alltrue([true, true, true])
true

> alltrue([true, false, true])
false

> alltrue([])
true  # vacuously true (empty list)
```

## anytrue()

Returns `true` if at least one element in the list is `true`:

```hcl
> anytrue([false, false, true])
true

> anytrue([false, false, false])
false

> anytrue([])
false  # no true elements
```

## Practical Use Cases

### Validating Multiple Conditions

```hcl
variable "subnet_ids" {
  type = list(string)
}

variable "min_subnets" {
  type    = number
  default = 2
}

locals {
  # Check all subnets start with "subnet-"
  all_valid_subnets = alltrue([
    for id in var.subnet_ids : startswith(id, "subnet-")
  ])
}

resource "null_resource" "validate" {
  lifecycle {
    precondition {
      condition     = local.all_valid_subnets
      error_message = "All subnet IDs must start with 'subnet-'."
    }
  }
}
```

### Checking Feature Flags

```hcl
variable "features" {
  type = object({
    enable_encryption  = bool
    enable_logging     = bool
    enable_monitoring  = bool
    enable_backup      = bool
  })
}

locals {
  feature_list     = values(var.features)
  all_features_on  = alltrue(local.feature_list)
  any_feature_on   = anytrue(local.feature_list)
  
  # Production requires all features enabled
  prod_ready = var.environment == "prod" ? alltrue(local.feature_list) : true
}

output "compliance_status" {
  value = local.all_features_on ? "Fully Compliant" : (
    local.any_feature_on ? "Partially Compliant" : "Non-Compliant"
  )
}
```

### Checking Security Group Rules

```hcl
variable "security_group_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    cidr_blocks = list(string)
  }))
}

locals {
  # Check if any rule allows access from the internet
  has_public_access = anytrue([
    for rule in var.security_group_rules :
    contains(rule.cidr_blocks, "0.0.0.0/0")
  ])
  
  # Production should not have public access
  is_secure_config = var.environment == "prod" ? !local.has_public_access : true
}
```

### Validating Tags Are Present

```hcl
variable "required_tag_keys" {
  type    = list(string)
  default = ["Environment", "Team", "CostCenter"]
}

variable "tags" {
  type = map(string)
}

locals {
  # Check all required tags are present
  all_tags_present = alltrue([
    for key in var.required_tag_keys : contains(keys(var.tags), key)
  ])
}

resource "null_resource" "tag_validation" {
  lifecycle {
    precondition {
      condition     = local.all_tags_present
      error_message = "All required tags must be present: ${join(", ", var.required_tag_keys)}."
    }
  }
}
```

## Combining with for Expressions

```hcl
variable "instance_types" {
  type    = list(string)
  default = ["t3.micro", "t3.small", "t3.medium"]
}

locals {
  # Check all are T-series instances
  all_t_series = alltrue([
    for t in var.instance_types : startswith(t, "t")
  ])
  
  # Check if any are large or xlarge
  any_large = anytrue([
    for t in var.instance_types : strcontains(t, "large")
  ])
}
```

## Conclusion

`alltrue()` and `anytrue()` are the programmatic equivalent of logical AND and OR across a list. Use them with `for` expressions to evaluate conditions across collections — validating that all resources meet requirements, checking whether any security rules violate policies, or testing feature flag combinations. They're particularly powerful in `precondition` and `postcondition` blocks for infrastructure validation.
