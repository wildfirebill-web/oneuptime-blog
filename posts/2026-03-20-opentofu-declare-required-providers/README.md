# How to Declare Required Providers in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Provider

Description: Learn how to declare required providers in OpenTofu using the required_providers block to specify provider sources and version constraints.

## Introduction

The `required_providers` block in a `terraform` block declares which providers your configuration needs, where to download them from, and which versions are acceptable. This declaration is essential for reproducible configurations and ensures everyone on your team uses the same provider version.

## Syntax

```hcl
terraform {
  required_providers {
    PROVIDER_LOCAL_NAME = {
      source  = "REGISTRY/NAMESPACE/TYPE"
      version = "VERSION_CONSTRAINT"
    }
  }
}
```

## Basic Declaration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## Multiple Providers

```hcl
terraform {
  required_version = ">= 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.20"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}
```

## Source Address Format

The `source` follows the format `HOSTNAME/NAMESPACE/TYPE`:

```hcl
# Default registry (registry.opentofu.org) - hostname can be omitted

aws = {
  source = "hashicorp/aws"        # Short form - uses registry.opentofu.org
}

# Explicit full address
aws = {
  source = "registry.opentofu.org/hashicorp/aws"
}

# Community providers from different namespaces
datadog = {
  source  = "DataDog/datadog"
  version = "~> 3.30"
}

# Private registry
internal = {
  source  = "registry.acme-corp.com/acme/internal"
  version = "~> 1.0"
}
```

## Why required_providers Matters

Without `required_providers`, OpenTofu makes assumptions:

```hcl
# BAD: no declaration - OpenTofu guesses the source and uses latest version
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
}

# GOOD: explicit declaration prevents "latest" surprises
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Constrained, not latest
    }
  }
}
```

## Version Constraint Best Practices

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"

      # Development: allow minor updates
      version = "~> 5.0"

      # Production: pin exact version for zero surprises
      # version = "= 5.31.0"

      # Minimum required version only (flexible, but less safe)
      # version = ">= 5.0"
    }
  }
}
```

## Running tofu init

After declaring required providers, initialize to download them:

```bash
tofu init

# Providers are downloaded to .terraform/providers/
# Lock file .terraform.lock.hcl is created or updated
```

## Checking Installed Providers

```bash
# List providers required by the configuration
tofu providers

# Output:
# Providers required by configuration:
# .
# └── provider[registry.opentofu.org/hashicorp/aws] ~> 5.0
```

## Conclusion

Declaring `required_providers` is a best practice that every OpenTofu configuration should follow. It documents dependencies, enforces version constraints, and prevents the configuration from accidentally using the wrong provider or an untested newer version. Always declare `required_providers` alongside `required_version` in a `versions.tf` file.
