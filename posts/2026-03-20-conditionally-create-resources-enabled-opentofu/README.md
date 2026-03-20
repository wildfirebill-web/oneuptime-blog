# How to Conditionally Create Resources with the enabled Meta-Argument in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Modules, Enabled, Conditional, HCL, Best Practices

Description: Learn how to implement an enabled variable pattern in OpenTofu modules to cleanly toggle entire feature sets on and off without changing module interfaces.

## Introduction

While OpenTofu doesn't have a built-in `enabled` meta-argument, the community convention is to use an `enabled` variable to control whether a module or resource group should be created. This pattern is cleaner than passing `count` or boolean flags directly to individual resources.

## Module-Level enabled Pattern

Define an `enabled` variable and use it as the basis for all `count` expressions within the module.

```hcl
# modules/monitoring/variables.tf

variable "enabled" {
  description = "Enable or disable all resources in this module"
  type        = bool
  default     = true
}

variable "service_name"    { type = string }
variable "alarm_actions"   { type = list(string); default = [] }
variable "cpu_threshold"   { type = number; default = 80 }
variable "mem_threshold"   { type = number; default = 90 }
```

```hcl
# modules/monitoring/main.tf
resource "aws_cloudwatch_metric_alarm" "cpu" {
  # All resources check the enabled variable via count
  count = var.enabled ? 1 : 0

  alarm_name          = "${var.service_name}-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = var.cpu_threshold
  alarm_actions       = var.alarm_actions
}

resource "aws_cloudwatch_metric_alarm" "memory" {
  count = var.enabled ? 1 : 0

  alarm_name          = "${var.service_name}-memory-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "MemoryUtilization"
  namespace           = "CWAgent"
  period              = 300
  statistic           = "Average"
  threshold           = var.mem_threshold
  alarm_actions       = var.alarm_actions
}

resource "aws_cloudwatch_dashboard" "service" {
  count          = var.enabled ? 1 : 0
  dashboard_name = "${var.service_name}-overview"
  dashboard_body = jsonencode({ /* ... */ })
}
```

## Calling the Module with enabled Control

```hcl
# Root module: enable monitoring in prod, disable in dev
module "monitoring" {
  source = "./modules/monitoring"

  enabled      = var.environment == "prod"
  service_name = "web-app"
  alarm_actions = var.environment == "prod" ? [aws_sns_topic.alerts.arn] : []
  cpu_threshold = 70
}
```

## enabled with Feature Sub-Modules

Organize optional features as sub-modules, each with their own enabled flag.

```hcl
module "waf" {
  source  = "./modules/waf"
  enabled = var.features.enable_waf

  alb_arn = aws_lb.app.arn
  rules   = var.waf_rules
}

module "shield" {
  source  = "./modules/shield"
  enabled = var.features.enable_shield_advanced

  resource_arn = aws_lb.app.arn
}

module "guardduty" {
  source  = "./modules/guardduty"
  enabled = var.features.enable_guardduty
}
```

## Combining enabled with count for Scaling

Use `enabled` to gate creation and `count` for scaling.

```hcl
variable "enabled"      { type = bool;   default = true }
variable "replica_count" { type = number; default = 1 }

resource "aws_instance" "worker" {
  # Zero instances when disabled; replica_count instances when enabled
  count = var.enabled ? var.replica_count : 0

  ami           = data.aws_ami.app.id
  instance_type = var.instance_type
  subnet_id     = element(var.subnet_ids, count.index % length(var.subnet_ids))

  tags = {
    Name  = "worker-${count.index + 1}"
    Index = count.index
  }
}
```

## Safe Outputs from enabled Modules

```hcl
# modules/monitoring/outputs.tf
output "dashboard_url" {
  description = "CloudWatch dashboard URL, or null if monitoring is disabled"
  value = var.enabled ? (
    "https://console.aws.amazon.com/cloudwatch/home#dashboards:name=${aws_cloudwatch_dashboard.service[0].dashboard_name}"
  ) : null
}

output "alarm_arns" {
  description = "List of created alarm ARNs"
  value = var.enabled ? [
    aws_cloudwatch_metric_alarm.cpu[0].arn,
    aws_cloudwatch_metric_alarm.memory[0].arn,
  ] : []
}
```

## Conclusion

The `enabled` variable pattern makes module interfaces self-documenting and easy to use. Module callers can disable entire feature sets with a single flag, and the module handles all the internal `count` logic. This is the recommended approach for optional infrastructure features in reusable module libraries.
