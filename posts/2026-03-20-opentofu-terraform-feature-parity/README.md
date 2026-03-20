# OpenTofu vs Terraform: Feature Parity and Differences

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Feature Parity, Comparison, Infrastructure as Code, Migration

Description: Understand what features OpenTofu and Terraform share, where they've diverged, and what OpenTofu-exclusive features are available - helping you evaluate the migration path and capabilities.

## Introduction

OpenTofu forked from Terraform 1.5.7 in 2023 when HashiCorp changed Terraform's license from MPL 2.0 to BSL. Since the fork, both projects have continued development independently. This guide covers the current feature parity and the areas where they've diverged.

## Shared Features (Complete Parity)

Everything from Terraform ≤ 1.5.7 works identically:

```hcl
# All of these work identically in both tools

# Resources, data sources, providers

resource "aws_instance" "web" { /* ... */ }
data "aws_ami" "latest" { /* ... */ }

# Variables, locals, outputs
variable "environment" { type = string }
locals { name_prefix = "${var.environment}-${var.project}" }
output "vpc_id" { value = aws_vpc.main.id }

# Modules
module "vpc" { source = "terraform-aws-modules/vpc/aws" }

# Count and for_each
resource "aws_subnet" "private" {
  count = 3
  cidr_block = cidrsubnet("10.0.0.0/16", 8, count.index)
}

# Dynamic blocks
dynamic "ingress" {
  for_each = var.ingress_rules
  content {
    from_port = ingress.value.from_port
    to_port   = ingress.value.to_port
    protocol  = ingress.value.protocol
  }
}

# Lifecycle rules
lifecycle {
  create_before_destroy = true
  prevent_destroy       = true
  ignore_changes        = [tags]
}

# Moved blocks
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}

# Check blocks (assertions)
check "health" {
  data "http" "endpoint" { url = "https://example.com/health" }
  assert {
    condition     = data.http.endpoint.status_code == 200
    error_message = "Health check failed"
  }
}
```

## OpenTofu-Exclusive Features

### 1. Provider Iteration (OpenTofu 1.8+)

```hcl
# Deploy to multiple regions without aliases
variable "aws_regions" {
  default = ["us-east-1", "eu-west-1", "ap-southeast-1"]
}

provider "aws" {
  for_each = toset(var.aws_regions)
  alias    = each.key
  region   = each.key
}

# Use all providers in a module
module "regional_vpc" {
  for_each = toset(var.aws_regions)
  source   = "./modules/vpc"
  providers = { aws = aws[each.key] }
}
```

### 2. Write-Only Attributes (OpenTofu 1.10+)

```hcl
# Passwords and secrets never stored in state
resource "aws_db_instance" "postgres" {
  identifier     = "prod-postgres"
  engine         = "postgres"
  instance_class = "db.t3.medium"

  # password is write-only: written to AWS, never stored in .tfstate
  password = var.db_password
}
```

### 3. Native State Encryption (OpenTofu 1.8+)

```hcl
terraform {
  encryption {
    key_provider "pbkdf2" "my_key" {
      passphrase = var.state_encryption_passphrase
    }

    method "aes_gcm" "default" {
      keys = key_provider.pbkdf2.my_key
    }

    state {
      method = method.aes_gcm.default
    }
  }
}
```

### 4. Loopable Import Blocks (OpenTofu 1.7+)

```hcl
# Import multiple resources with for_each
import {
  for_each = var.existing_bucket_names
  to       = aws_s3_bucket.existing[each.key]
  id       = each.value
}
```

## Terraform-Exclusive Features (BUSL-Licensed)

These features exist in Terraform but not in OpenTofu:

| Feature | Terraform | OpenTofu |
|---------|-----------|----------|
| Stacks (preview) | Yes (BSL) | No |
| Cloud workspaces | TF Cloud only | No |
| Sentinel policies | TF Enterprise | OPA alternative |

## Version Alignment

| OpenTofu | Terraform Equivalent | Key Additions |
|----------|---------------------|---------------|
| 1.6.x | ~1.6.x | Fork stabilization |
| 1.7.x | N/A | Loopable import blocks |
| 1.8.x | N/A | Provider iteration, native state encryption |
| 1.9.x | N/A | Variable validation improvements |
| 1.10.x | N/A | Write-only attributes, ephemeral resources |

## State File Compatibility

```bash
# OpenTofu and Terraform state files are identical format (v4)
# You can switch between tools without state migration

# Verify format
cat .terraform/terraform.tfstate | jq '.version'
# Both output: 4
```

## Conclusion

OpenTofu maintains full compatibility with Terraform ≤ 1.5.7 and has since added exclusive features: provider iteration, write-only attributes, native state encryption, and loopable import blocks. Terraform has continued development under BSL with its own features like Stacks. For open-source licensing requirements or access to the OpenTofu-exclusive features, OpenTofu is the clear choice. For organizations already invested in Terraform Enterprise and Terraform Cloud, the migration decision depends on licensing preferences and which feature set better matches requirements.
