# How to Use the enabled Meta-Argument with Data Sources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Data Source, Enabled, Count, Conditional, HCL, Infrastructure as Code

Description: Learn how to conditionally enable or disable data sources in OpenTofu using the count trick and the enabled pattern for conditional data source evaluation.

---

OpenTofu doesn't have a built-in `enabled` meta-argument for data sources, but you can conditionally include or skip data sources using `count = var.enabled ? 1 : 0`. This is a common pattern for conditionally querying infrastructure based on variables or environment configuration.

---

## The count = 0/1 Pattern

Set `count` to `1` to include the data source, or `0` to skip it entirely:

```hcl
variable "use_existing_vpc" {
  type    = bool
  default = false
}

# Only query the VPC if use_existing_vpc is true

data "aws_vpc" "existing" {
  count = var.use_existing_vpc ? 1 : 0

  tags = {
    Name = "existing-vpc"
  }
}

# Pick between the existing VPC or a new one
locals {
  vpc_id = var.use_existing_vpc ? data.aws_vpc.existing[0].id : aws_vpc.new[0].id
}
```

---

## Conditional Secret Lookup

```hcl
variable "environment" {
  type    = string
  default = "development"
}

# Only read production secrets when in production
data "aws_secretsmanager_secret_version" "prod_db" {
  count     = var.environment == "production" ? 1 : 0
  secret_id = "production/database/password"
}

locals {
  db_password = var.environment == "production" ? (
    jsondecode(data.aws_secretsmanager_secret_version.prod_db[0].secret_string)["password"]
  ) : var.dev_db_password
}
```

---

## Conditionally Reading Certificates

```hcl
variable "enable_https" {
  type    = bool
  default = true
}

# Only look up the ACM certificate if HTTPS is enabled
data "aws_acm_certificate" "app" {
  count    = var.enable_https ? 1 : 0
  domain   = "app.example.com"
  statuses = ["ISSUED"]
}

resource "aws_lb_listener" "https" {
  count = var.enable_https ? 1 : 0

  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = data.aws_acm_certificate.app[0].arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

---

## Accessing Results Safely

When using `count`, always access results with the `[0]` index inside conditional expressions to avoid errors:

```hcl
variable "use_existing_key" {
  type    = bool
  default = false
}

data "aws_key_pair" "existing" {
  count    = var.use_existing_key ? 1 : 0
  key_name = "my-existing-key"
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  # Use try() to safely access the optional data source result
  key_name = var.use_existing_key ? data.aws_key_pair.existing[0].key_name : null
}
```

---

## Using for_each as an Enable/Disable Pattern

An alternative to `count` that's clearer for some use cases:

```hcl
variable "enabled_regions" {
  type    = set(string)
  default = ["us-east-1", "us-west-2"]
}

# Only query zones in enabled regions
data "aws_availability_zones" "available" {
  for_each = var.enabled_regions
  provider = aws.regions[each.key]

  state = "available"
}
```

---

## Pattern Summary

| Pattern | Use When |
|---|---|
| `count = var.enabled ? 1 : 0` | Single optional data source |
| `count = length(var.list) > 0 ? 1 : 0` | Data source needed only if list is non-empty |
| `for_each = var.enabled ? toset(["main"]) : toset([])` | Named optional data source |

---

## Summary

OpenTofu data sources support `count` and `for_each` meta-arguments, which you can use to conditionally enable or disable data source evaluation. The `count = condition ? 1 : 0` pattern is the standard way to make a data source optional. Always access conditionally enabled data sources with `[0]` or guard with `try()` to handle the case where the data source is disabled.
