# How to Use the contains Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Contains, List Functions, HCL, Infrastructure as Code, DevOps

Description: Learn how to use the contains function in OpenTofu to check whether a list or set includes a specific value.

---

`contains()` returns `true` if the given list or set contains the given value, and `false` otherwise.

---

## Syntax

```hcl
contains(list, value)
```

---

## Basic Examples

```hcl
locals {
  fruits    = ["apple", "banana", "cherry"]
  has_apple  = contains(local.fruits, "apple")   # true
  has_mango  = contains(local.fruits, "mango")   # false

  numbers    = [1, 2, 3, 4, 5]
  has_three  = contains(local.numbers, 3)        # true
  has_ten    = contains(local.numbers, 10)       # false
}
```

---

## Validating Variable Values

```hcl
variable "instance_type" {
  type = string

  validation {
    condition = contains([
      "t3.micro", "t3.small", "t3.medium",
      "m5.large", "m5.xlarge"
    ], var.instance_type)

    error_message = "instance_type must be one of: t3.micro, t3.small, t3.medium, m5.large, m5.xlarge."
  }
}

variable "region" {
  type = string

  validation {
    condition     = contains(["us-east-1", "us-west-2", "eu-west-1"], var.region)
    error_message = "region must be us-east-1, us-west-2, or eu-west-1."
  }
}
```

---

## Conditional Resource Creation

```hcl
variable "enabled_features" {
  type    = list(string)
  default = ["monitoring", "backups"]
}

# Only create CloudWatch alarms if monitoring is enabled

resource "aws_cloudwatch_metric_alarm" "cpu" {
  count = contains(var.enabled_features, "monitoring") ? 1 : 0

  alarm_name          = "cpu-utilization"
  comparison_operator = "GreaterThanThreshold"
  threshold           = 80
  # ...
}

# Only create automated backups if backups is enabled
resource "aws_backup_plan" "daily" {
  count = contains(var.enabled_features, "backups") ? 1 : 0

  name = "daily-backup"
  # ...
}
```

---

## Filtering Lists with contains

```hcl
variable "all_instances" {
  type = list(object({
    id   = string
    type = string
    tags = list(string)
  }))
}

variable "required_tags" {
  type    = list(string)
  default = ["production", "monitored"]
}

locals {
  # Find instances that have the "production" tag
  production_instances = [
    for instance in var.all_instances :
    instance
    if contains(instance.tags, "production")
  ]
}
```

---

## Security Group Source Checking

```hcl
variable "allowed_sg_ids" {
  type    = list(string)
  default = ["sg-111", "sg-222"]
}

locals {
  new_sg_id = aws_security_group.app.id

  # Check if the new SG ID is already in the allowed list
  already_allowed = contains(var.allowed_sg_ids, local.new_sg_id)
}
```

---

## contains vs index

```hcl
locals {
  my_list = ["a", "b", "c"]

  # contains: just checks presence
  has_b    = contains(local.my_list, "b")      # true

  # index: gets the position (errors if not found)
  pos_b    = index(local.my_list, "b")         # 1

  # Safe index check
  safe_pos = contains(local.my_list, "b") ? index(local.my_list, "b") : -1
}
```

---

## Summary

`contains(list, value)` checks if a list or set includes a specific value, returning a boolean. It's most commonly used in validation blocks to restrict variable values to an allowed set, in conditional expressions to enable or disable features based on a list of enabled flags, and in for expression filters to select items that match membership criteria.
