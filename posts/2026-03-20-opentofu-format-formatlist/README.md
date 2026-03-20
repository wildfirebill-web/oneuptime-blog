# How to Use format() and formatlist() in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Functions, String

Description: Learn how to use the format() and formatlist() functions in OpenTofu for sprintf-style string formatting.

`format()` produces a formatted string using printf-style directives, while `formatlist()` applies the same formatting to a list of values. They give you precise control over string output.

## format() Syntax

```hcl
format(format_string, values...)
```

## Format Verbs

```hcl
# %s - string

> format("Hello, %s!", "world")
"Hello, world!"

# %d - decimal integer
> format("Port: %d", 8080)
"Port: 8080"

# %f - float
> format("%.2f GB", 1.5)
"1.50 GB"

# %05d - zero-padded integer
> format("%05d", 42)
"00042"

# %q - quoted string
> format("Name: %q", "my resource")
"Name: \"my resource\""

# %v - value in default format
> format("Value: %v", true)
"Value: true"

# %% - literal percent sign
> format("Progress: %d%%", 75)
"Progress: 75%"
```

## Practical format() Examples

```hcl
variable "environment" { default = "prod" }
variable "region"      { default = "us-east-1" }
variable "index"       { type = number }

locals {
  # Zero-padded resource names
  server_name = format("web-%03d", var.index)
  # "web-001", "web-002", etc.
  
  # Composite names
  bucket_name = format("%s-%s-data", var.environment, var.region)
  # "prod-us-east-1-data"
  
  # Log format
  log_prefix = format("[%s][%s]", upper(var.environment), var.region)
  # "[PROD][us-east-1]"
}
```

## formatlist()

Applies format to each element of a list:

```hcl
# formatlist(format_string, list)
> formatlist("Hello, %s!", ["Alice", "Bob", "Charlie"])
["Hello, Alice!", "Hello, Bob!", "Hello, Charlie!"]

> formatlist("arn:aws:iam::%s:root", ["111111111111", "222222222222"])
["arn:aws:iam::111111111111:root", "arn:aws:iam::222222222222:root"]
```

## Generating ARNs with formatlist()

```hcl
variable "account_ids" {
  type    = list(string)
  default = ["111111111111", "222222222222", "333333333333"]
}

variable "role_name" {
  type    = string
  default = "TerraformDeployRole"
}

locals {
  role_arns = formatlist(
    "arn:aws:iam::%s:role/${var.role_name}",
    var.account_ids
  )
  # [
  #   "arn:aws:iam::111111111111:role/TerraformDeployRole",
  #   "arn:aws:iam::222222222222:role/TerraformDeployRole",
  #   "arn:aws:iam::333333333333:role/TerraformDeployRole",
  # ]
}

data "aws_iam_policy_document" "assume_role" {
  statement {
    principals {
      type        = "AWS"
      identifiers = local.role_arns
    }
    actions = ["sts:AssumeRole"]
  }
}
```

## Building Multiple Resource Names

```hcl
variable "services" {
  type    = list(string)
  default = ["api", "worker", "scheduler"]
}

variable "environment" {
  default = "prod"
}

locals {
  sg_names     = formatlist("${var.environment}-%s-sg", var.services)
  bucket_names = formatlist("myapp-${var.environment}-%s-data", var.services)
}

resource "aws_security_group" "services" {
  count = length(local.sg_names)
  name  = local.sg_names[count.index]
  # "prod-api-sg", "prod-worker-sg", "prod-scheduler-sg"
}
```

## Conclusion

`format()` gives you sprintf-style control over string construction, and `formatlist()` applies that formatting across entire lists at once. Use them for generating consistent resource names, building ARNs from account IDs, creating numbered resource names with zero-padding, and any situation where you need precise string composition.
