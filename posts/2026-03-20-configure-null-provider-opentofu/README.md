# How to Configure Null Provider with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Provider, Automation, DevOps

Description: Learn how to configure and use the Null provider in OpenTofu to manage Null resources as code.

## Introduction

The Null provider for OpenTofu enables managing Null resources with the same plan/apply workflow as your cloud infrastructure. This guide covers authentication, basic resource configuration, and production best practices.

## Provider Installation

```hcl
terraform {
  required_providers {
    null = {
      source  = "hashicorp/null"
      version = "~> 3.0"
    }
  }
  required_version = ">= 1.6.0"
}
```

## Provider Configuration

The null provider requires no configuration — it has no external service to authenticate with:

```hcl
provider "null" {}
```

## null_resource Example

`null_resource` runs local-exec or remote-exec provisioners without creating a real infrastructure resource:

```hcl
resource "null_resource" "run_script" {
  # Re-run triggers when this value changes
  triggers = {
    script_hash = filemd5("${path.module}/scripts/bootstrap.sh")
  }

  provisioner "local-exec" {
    command = "bash ${path.module}/scripts/bootstrap.sh"
  }
}
```

## Variables

```hcl
variable "name"        { type = string }
variable "environment" { type = string }
```

## Outputs

```hcl
output "null_resource_id" { value = null_resource.run_script.id }
```

## Best Practices

- Use `triggers` to control when a `null_resource` re-runs its provisioners
- Prefer dedicated provider resources over `null_resource` with provisioners where possible
- Commit the `.terraform.lock.hcl` file to lock exact provider versions

## Conclusion

The `null` provider's primary use case is running provisioners tied to the OpenTofu lifecycle. Use `null_resource` with `triggers` to re-run scripts when specific inputs change, such as a file hash or a version string.
