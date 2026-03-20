# How to Use for_each with Modules in OpenTofu - Foreach

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Modules, for_each

Description: Learn how to use for_each with modules in OpenTofu to create multiple instances of a module from a map or set of strings.

When you need to create multiple instances of a module based on a map or set of values, OpenTofu's `for_each` meta-argument provides a powerful and flexible approach. Unlike `count`, `for_each` lets you reference each instance by a meaningful key, making your infrastructure easier to manage.

## Basic Syntax

```hcl
module "example" {
  source   = "./modules/example"
  for_each = var.instances

  name = each.key
  size = each.value
}
```

Inside the module call, `each.key` holds the map key and `each.value` holds the corresponding value.

## Using for_each with a Set of Strings

```hcl
variable "regions" {
  type    = set(string)
  default = ["us-east-1", "us-west-2", "eu-west-1"]
}

module "vpc" {
  source   = "./modules/vpc"
  for_each = var.regions

  region     = each.key
  cidr_block = "10.0.0.0/16"
}
```

## Using for_each with a Map

```hcl
variable "environments" {
  type = map(object({
    instance_type = string
    min_size      = number
    max_size      = number
  }))
  default = {
    dev = {
      instance_type = "t3.small"
      min_size      = 1
      max_size      = 3
    }
    staging = {
      instance_type = "t3.medium"
      min_size      = 2
      max_size      = 5
    }
    prod = {
      instance_type = "t3.large"
      min_size      = 3
      max_size      = 10
    }
  }
}

module "app_cluster" {
  source   = "./modules/ecs_cluster"
  for_each = var.environments

  environment   = each.key
  instance_type = each.value.instance_type
  min_size      = each.value.min_size
  max_size      = each.value.max_size
}
```

## Accessing Module Outputs with for_each

When a module uses `for_each`, its outputs become a map keyed by `each.key`:

```hcl
module "database" {
  source   = "./modules/rds"
  for_each = var.environments

  env         = each.key
  db_password = var.db_passwords[each.key]
}

output "database_endpoints" {
  value = {
    for env, db in module.database :
    env => db.endpoint
  }
}

output "primary_db_endpoint" {
  value = module.database["prod"].endpoint
}
```

## Practical Example: Multi-Environment Deployment

```hcl
# variables.tf

variable "services" {
  type = map(object({
    image      = string
    port       = number
    replicas   = number
    cpu        = number
    memory     = number
  }))
}

# main.tf
module "service" {
  source   = "./modules/k8s_service"
  for_each = var.services

  name     = each.key
  image    = each.value.image
  port     = each.value.port
  replicas = each.value.replicas
  cpu      = each.value.cpu
  memory   = each.value.memory

  namespace = var.namespace
  labels = {
    app     = each.key
    env     = var.environment
    version = each.value.image
  }
}

# terraform.tfvars
services = {
  frontend = {
    image    = "nginx:1.25"
    port     = 80
    replicas = 3
    cpu      = 250
    memory   = 512
  }
  api = {
    image    = "myapp-api:2.1"
    port     = 8080
    replicas = 5
    cpu      = 500
    memory   = 1024
  }
  worker = {
    image    = "myapp-worker:2.1"
    port     = 9090
    replicas = 2
    cpu      = 1000
    memory   = 2048
  }
}
```

## Converting Lists to Maps for for_each

Since `for_each` requires a map or set, convert lists using `for` expressions:

```hcl
variable "bucket_names" {
  type    = list(string)
  default = ["logs", "backups", "artifacts"]
}

module "s3_bucket" {
  source   = "./modules/s3"
  for_each = toset(var.bucket_names)

  bucket_name = each.key
  tags = {
    Purpose = each.key
  }
}
```

## Conclusion

`for_each` with modules is ideal when you need named instances, want to add or remove specific instances without affecting others, or when your module instances differ significantly in their configuration. It produces more stable plans than `count` when elements are inserted or removed from the middle of a list.
