# How to Use the strcontains Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the strcontains function in OpenTofu to check if a string contains a specific substring for conditional resource configuration.

## Introduction

The `strcontains` function in OpenTofu returns `true` if a given string contains a specified substring, and `false` otherwise. It is a simple but powerful function for conditional logic, validation, and filtering in your infrastructure configurations.

## Syntax

```hcl
strcontains(string, substr)
```

- **string** — the string to search within
- **substr** — the substring to search for
- Returns a boolean (`true` or `false`)

## Basic Examples

```hcl
output "contains_true" {
  value = strcontains("hello world", "world")  # Returns true
}

output "contains_false" {
  value = strcontains("hello world", "foo")    # Returns false
}

output "empty_substr" {
  value = strcontains("hello", "")             # Returns true (empty string is always contained)
}

output "case_sensitive" {
  value = strcontains("Hello", "hello")        # Returns false (case-sensitive)
}
```

## Practical Use Cases

### Conditional Resource Configuration

```hcl
variable "instance_type" {
  type    = string
  default = "m5.xlarge"
}

locals {
  # Check if this is a memory-optimized instance
  is_memory_optimized = strcontains(var.instance_type, "r5") || strcontains(var.instance_type, "r6i")
  # Check if this is a GPU instance
  is_gpu_instance = strcontains(var.instance_type, "p3") || strcontains(var.instance_type, "g4")
}

resource "aws_instance" "compute" {
  ami           = local.is_gpu_instance ? data.aws_ami.gpu.id : data.aws_ami.standard.id
  instance_type = var.instance_type

  tags = {
    Name            = "compute-node"
    MemoryOptimized = tostring(local.is_memory_optimized)
    GpuEnabled      = tostring(local.is_gpu_instance)
  }
}
```

### Filtering Resources by Name Pattern

```hcl
variable "subnet_names" {
  type = list(string)
  default = [
    "public-subnet-1",
    "private-subnet-1",
    "public-subnet-2",
    "private-subnet-2",
    "db-subnet-1"
  ]
}

locals {
  # Filter to only public subnets
  public_subnets  = [for s in var.subnet_names : s if strcontains(s, "public")]
  private_subnets = [for s in var.subnet_names : s if strcontains(s, "private")]
}

output "public_subnets" {
  value = local.public_subnets  # ["public-subnet-1", "public-subnet-2"]
}
```

### Input Validation

```hcl
variable "docker_image" {
  type        = string
  description = "Docker image tag (must include a registry hostname)"

  validation {
    condition     = strcontains(var.docker_image, "/")
    error_message = "docker_image must include a registry hostname (e.g., registry.example.com/myapp:latest)."
  }

  validation {
    condition     = strcontains(var.docker_image, ":")
    error_message = "docker_image must include a tag (e.g., myapp:v1.0.0)."
  }
}
```

### Detecting Production Resources

```hcl
variable "bucket_name" {
  type    = string
  default = "prod-app-data"
}

locals {
  is_production = strcontains(lower(var.bucket_name), "prod")
}

resource "aws_s3_bucket" "data" {
  bucket = var.bucket_name

  lifecycle {
    prevent_destroy = local.is_production  # Protect production buckets
  }
}
```

### Tag-Based Filtering

```hcl
variable "resources" {
  type = map(object({
    name = string
    tags = list(string)
  }))
}

locals {
  # Find resources tagged as "critical"
  critical_resources = {
    for k, v in var.resources :
    k => v if contains(v.tags, "critical")
  }

  # Find resources with "legacy" in the name
  legacy_resources = {
    for k, v in var.resources :
    k => v if strcontains(lower(v.name), "legacy")
  }
}
```

## Step-by-Step Usage

1. Identify a substring you want to search for.
2. Call `strcontains(string, substr)`.
3. Use the boolean result in conditionals, `for` expressions, or `validation` blocks.
4. Test in `tofu console`:

```bash
tofu console

> strcontains("us-east-1", "east")
true
> strcontains("prod-database", "staging")
false
```

## Case-Insensitive Matching

`strcontains` is case-sensitive. For case-insensitive matching, normalize case first:

```hcl
locals {
  input = "PRODUCTION-DB"
  is_prod = strcontains(lower(local.input), "production")  # true
}
```

## Conclusion

The `strcontains` function is a clean, readable way to test for substring presence in OpenTofu. Use it for conditional resource configuration, input validation, list filtering, and detecting patterns in resource names and identifiers. For case-insensitive matches, pair it with `lower()` or `upper()`.
