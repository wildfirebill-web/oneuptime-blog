# How to Use the tobool Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the tobool function in OpenTofu to convert string and null values to boolean type for type-safe conditionals.

## Introduction

The `tobool` function in OpenTofu converts a value to boolean type. It is useful when you receive boolean values as strings (e.g., `"true"` or `"false"`) from external data sources, CSV files, or environment variables, and need to use them as actual booleans in conditionals.

## Syntax

```hcl
tobool(value)
```

- Accepts `true`, `false`, `"true"`, `"false"`, `null`
- Returns a boolean value
- Raises an error for other string values

## Basic Examples

```hcl
output "from_string_true" {
  value = tobool("true")   # Returns true
}

output "from_string_false" {
  value = tobool("false")  # Returns false
}

output "from_bool" {
  value = tobool(true)     # Returns true
}

output "from_null" {
  value = tobool(null)     # Returns null
}
```

## Practical Use Cases

### Processing CSV Boolean Flags

```hcl
locals {
  instances = csvdecode(file("${path.module}/data/instances.csv"))
  # CSV has: name,instance_type,public_ip
  # Values: "web-1","t3.medium","true"
}

resource "aws_instance" "servers" {
  for_each      = { for i in local.instances : i.name => i }
  ami           = data.aws_ami.ubuntu.id
  instance_type = each.value.instance_type

  # CSV values are strings, convert to bool for associate_public_ip_address
  associate_public_ip_address = tobool(each.value.public_ip)
}
```

### External Data Source Boolean

```hcl
data "external" "config" {
  program = ["bash", "-c", "echo '{\"debug\": \"true\"}'"]
}

locals {
  debug_mode = tobool(data.external.config.result["debug"])
}

resource "aws_lambda_function" "api" {
  filename      = "function.zip"
  function_name = "api"
  role          = aws_iam_role.lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs18.x"

  environment {
    variables = {
      DEBUG = tostring(local.debug_mode)
    }
  }
}
```

### Variable Type Normalization

```hcl
variable "enable_feature_raw" {
  type    = string
  default = "true"
}

locals {
  enable_feature = tobool(var.enable_feature_raw)
}

resource "aws_cloudwatch_log_group" "feature_logs" {
  count             = local.enable_feature ? 1 : 0
  name              = "/app/feature-logs"
  retention_in_days = 7
}
```

## Step-by-Step Usage

```bash
tofu console

> tobool("true")
true
> tobool(false)
false
> tobool(null)
null
```

## Conclusion

The `tobool` function provides explicit type conversion from string to boolean in OpenTofu. Use it when processing CSV data, external source outputs, or string-typed variables that represent boolean flags. This ensures type safety in conditionals and resource configuration.
