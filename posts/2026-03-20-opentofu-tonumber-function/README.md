# How to Use the tonumber Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the tonumber function in OpenTofu to convert string values to numbers for arithmetic operations with CSV and external data.

## Introduction

The `tonumber` function in OpenTofu converts a string representation of a number to an actual numeric value. This is essential when processing CSV data, external data sources, or SSM parameters where numeric values are returned as strings.

## Syntax

```hcl
tonumber(value)
```

- Accepts numeric strings (`"42"`, `"3.14"`) and numbers
- Returns a number
- Raises an error for non-numeric strings

## Basic Examples

```hcl
output "from_string" {
  value = tonumber("42")     # Returns 42
}

output "from_float_string" {
  value = tonumber("3.14")   # Returns 3.14
}

output "from_number" {
  value = tonumber(100)      # Returns 100 (no-op)
}
```

## Practical Use Cases

### Processing CSV Data

```hcl
locals {
  instances = csvdecode(<<-CSV
    name,instance_type,replica_count,disk_gb
    api,t3.medium,3,50
    worker,t3.small,5,20
  CSV
  )
}

resource "aws_autoscaling_group" "services" {
  for_each = { for i in local.instances : i.name => i }

  min_size         = tonumber(each.value.replica_count)
  max_size         = tonumber(each.value.replica_count) * 2
  desired_capacity = tonumber(each.value.replica_count)

  vpc_zone_identifier = var.subnet_ids

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
}
```

### Reading Numeric Parameters from SSM

```hcl
data "aws_ssm_parameter" "max_connections" {
  name = "/app/db/max-connections"
}

locals {
  # SSM parameters are always strings
  max_conn = tonumber(data.aws_ssm_parameter.max_connections.value)
}

resource "aws_db_parameter_group" "app" {
  family = "postgres14"
  name   = "app-params"

  parameter {
    name  = "max_connections"
    value = tostring(local.max_conn)
  }
}
```

### External Data Numeric Fields

```hcl
data "external" "cluster_size" {
  program = ["bash", "-c", "echo '{\"nodes\": \"5\", \"cpu\": \"16\"}'"]
}

locals {
  node_count = tonumber(data.external.cluster_size.result["nodes"])
  total_cpu  = tonumber(data.external.cluster_size.result["cpu"]) * local.node_count
}

output "total_cluster_cpu" {
  value = local.total_cpu  # 80
}
```

## Step-by-Step Usage

```bash
tofu console

> tonumber("100")
100
> tonumber("3.14")
3.14
> tonumber(42) + 8
50
```

## Conclusion

The `tonumber` function is essential when OpenTofu receives numeric data as strings - from CSV files, SSM parameters, external data sources, and environment variables. It enables you to perform arithmetic on string-typed numeric data from dynamic sources.
