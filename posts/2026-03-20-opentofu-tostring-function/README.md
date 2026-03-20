# How to Use the tostring Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the tostring function in OpenTofu to convert numbers, booleans, and null values to string type for use in resource attributes.

## Introduction

The `tostring` function in OpenTofu converts a value to string type. Many resource attributes only accept strings, so you need `tostring` to convert numbers, booleans, and null values when assigning them to string-typed fields.

## Syntax

```hcl
tostring(value)
```

- Converts numbers, booleans, and null to strings
- Returns the value unchanged if already a string

## Basic Examples

```hcl
output "number_to_string" {
  value = tostring(42)      # Returns "42"
}

output "bool_to_string" {
  value = tostring(true)    # Returns "true"
}

output "float_to_string" {
  value = tostring(3.14)    # Returns "3.14"
}

output "null_to_string" {
  value = tostring(null)    # Returns null
}
```

## Practical Use Cases

Resource Tags (Must Be Strings)

```hcl
variable "instance_count" {
  type    = number
  default = 5
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"

  tags = {
    Name              = "app-server"
    # Tags must be strings - convert numbers and bools
    InstanceCount     = tostring(var.instance_count)
    MonitoringEnabled = tostring(var.enable_monitoring)
    PortNumber        = tostring(8080)
  }
}
```

### Database Parameter Groups

```hcl
resource "aws_db_parameter_group" "postgres" {
  family = "postgres14"
  name   = "myapp-params"

  parameter {
    name  = "max_connections"
    # Parameter value must be a string
    value = tostring(var.max_connections)
  }

  parameter {
    name  = "shared_buffers"
    value = tostring(var.shared_buffers_mb)
  }
}
```

### CloudWatch Metric Alarm Threshold

```hcl
variable "cpu_threshold" {
  type    = number
  default = 80
}

resource "aws_cloudwatch_metric_alarm" "cpu" {
  alarm_name          = "high-cpu"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  statistic           = "Average"
  period              = 60
  threshold           = var.cpu_threshold  # number is OK here
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2

  alarm_description = "CPU usage exceeded ${tostring(var.cpu_threshold)}%"
}
```

## Step-by-Step Usage

```bash
tofu console

> tostring(42)
"42"
> tostring(true)
"true"
> "${42}"  # Interpolation also converts
"42"
```

## tostring vs String Interpolation

```hcl
# Both produce the same result:

tostring(42)
"${42}"

# tostring is explicit and clearer for type conversion
```

## Conclusion

The `tostring` function provides explicit type conversion to string in OpenTofu. Use it when assigning numeric or boolean values to string-typed resource attributes like tags, database parameters, and alarm descriptions. It makes type conversions explicit and readable.
