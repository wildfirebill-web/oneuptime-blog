# How to Use the zipmap Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, zipmap, Map Functions, HCL, Infrastructure as Code, DevOps

Description: Learn how to use the zipmap function in OpenTofu to create a map from two parallel lists of keys and values.

---

`zipmap()` constructs a map from two lists: a list of keys and a list of values. Elements at the same index are paired together. The lists must be the same length.

---

## Syntax

```hcl
zipmap(keys_list, values_list)
```

---

## Basic Examples

```hcl
locals {
  names  = ["alice", "bob", "charlie"]
  roles  = ["admin", "user", "developer"]

  user_role_map = zipmap(local.names, local.roles)
  # {
  #   "alice"   = "admin"
  #   "bob"     = "user"
  #   "charlie" = "developer"
  # }
}
```

---

## Creating Instance Type Map

```hcl
variable "instance_names" {
  type    = list(string)
  default = ["web", "api", "worker"]
}

variable "instance_types" {
  type    = list(string)
  default = ["t3.micro", "t3.small", "t3.medium"]
}

locals {
  # Create a map of name → type
  instance_config = zipmap(var.instance_names, var.instance_types)
  # { "web" = "t3.micro", "api" = "t3.small", "worker" = "t3.medium" }
}

resource "aws_instance" "app" {
  for_each = local.instance_config

  ami           = data.aws_ami.amazon_linux.id
  instance_type = each.value   # "t3.micro", "t3.small", or "t3.medium"

  tags = { Name = each.key }
}
```

---

## Combining Resource Outputs into a Map

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  tags = { Name = "web-${count.index}" }
}

locals {
  # Create a map of instance name → ID
  instance_map = zipmap(
    aws_instance.web[*].tags["Name"],   # ["web-0", "web-1", "web-2"]
    aws_instance.web[*].id             # ["i-111", "i-222", "i-333"]
  )
  # { "web-0" = "i-111", "web-1" = "i-222", "web-2" = "i-333" }
}
```

---

## Creating Tag Maps from Lists

```hcl
variable "tag_names" {
  type    = list(string)
  default = ["Environment", "Team", "Project"]
}

variable "tag_values" {
  type    = list(string)
  default = ["production", "platform", "myapp"]
}

resource "aws_s3_bucket" "app" {
  bucket = "myapp-data"
  tags   = zipmap(var.tag_names, var.tag_values)
  # { "Environment" = "production", "Team" = "platform", "Project" = "myapp" }
}
```

---

## Building Environment Variable Maps

```hcl
variable "env_var_names" {
  type    = list(string)
  default = ["APP_PORT", "APP_ENV", "LOG_LEVEL"]
}

variable "env_var_values" {
  type    = list(string)
  default = ["8080", "production", "warn"]
}

locals {
  env_vars = zipmap(var.env_var_names, var.env_var_values)
}

resource "aws_ecs_task_definition" "app" {
  family = "app"

  container_definitions = jsonencode([{
    name = "app"
    environment = [
      for k, v in local.env_vars : { name = k, value = v }
    ]
  }])
}
```

---

## zipmap vs for Expression

```hcl
locals {
  names  = ["a", "b", "c"]
  values = [1, 2, 3]

  # These produce equivalent maps:
  using_zipmap = zipmap(local.names, local.values)
  using_for    = { for i, name in local.names : name => local.values[i] }
}
```

`zipmap()` is more concise. Use a for expression when you need to transform or filter the values during construction.

---

## Summary

`zipmap(keys, values)` builds a map by pairing elements from two lists at the same index. Use it to construct maps from parallel lists — instance names and types, tag names and values, or environment variable names and values. It's the clean way to merge two correlated lists into a single lookup map. Both lists must be the same length.
