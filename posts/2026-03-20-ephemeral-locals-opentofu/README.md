# How to Use Ephemeral Locals in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Local, Ephemeral, Security, Infrastructure as Code, DevOps

Description: A guide to using ephemeral local values in OpenTofu to compute and pass temporary values without storing them in state.

## Introduction

Ephemeral locals in OpenTofu (introduced in 1.11) are local values derived from ephemeral sources (like ephemeral variables or resources). Like ephemeral variables, ephemeral locals are not stored in the state file, making them safe for working with temporary credentials and sensitive computed values.

## Declaring Ephemeral Locals

```hcl
# Ephemeral variable (source of ephemeral data)

variable "vault_token" {
  type      = string
  ephemeral = true
  sensitive = true
}

# Ephemeral local derived from ephemeral variable
locals {
  # This local inherits ephemeral nature from its source
  auth_header = "Bearer ${var.vault_token}"

  # Complex computation with ephemeral data
  api_config = {
    url     = "https://api.example.com"
    token   = var.vault_token
    timeout = 30
  }
}
```

## Ephemeral Propagation

When a local value references an ephemeral source, it automatically becomes ephemeral:

```hcl
variable "aws_session_token" {
  type      = string
  ephemeral = true
  sensitive = true
}

locals {
  # This local is automatically ephemeral because it references
  # an ephemeral variable
  aws_credentials = {
    access_key    = var.aws_access_key_id    # ephemeral
    secret_key    = var.aws_secret_access_key # ephemeral
    session_token = var.aws_session_token     # ephemeral
  }

  # This CANNOT be used in regular resource attributes
  # It can only be used in ephemeral-compatible contexts
  # (providers, provisioner connections, etc.)
}
```

## Using Ephemeral Locals in Providers

```hcl
variable "vault_role_id" {
  type      = string
  ephemeral = true
}

variable "vault_secret_id" {
  type      = string
  ephemeral = true
  sensitive = true
}

locals {
  # Compute vault auth token from ephemeral credentials
  # This is ephemeral because its sources are ephemeral
  vault_auth = {
    role_id   = var.vault_role_id
    secret_id = var.vault_secret_id
  }
}

# Use ephemeral local in provider configuration
provider "vault" {
  address = "https://vault.example.com"

  auth_login_approle {
    role_id   = local.vault_auth.role_id
    secret_id = local.vault_auth.secret_id
  }
}
```

## Practical Example: Dynamic Configuration

```hcl
# Get temporary credentials from an ephemeral resource
ephemeral "aws_temporary_credentials" "deploy" {
  role_arn    = "arn:aws:iam::123456789012:role/DeployRole"
  duration    = "1h"
}

locals {
  # Build AWS credential configuration (ephemeral)
  aws_creds = {
    access_key    = ephemeral.aws_temporary_credentials.deploy.access_key
    secret_key    = ephemeral.aws_temporary_credentials.deploy.secret_key
    session_token = ephemeral.aws_temporary_credentials.deploy.session_token
  }

  # Build API endpoint configuration
  deploy_config = {
    credentials = local.aws_creds
    region      = var.aws_region
    account_id  = var.aws_account_id
  }
}
```

## Limitations

```hcl
# Ephemeral locals CANNOT be used in:
# 1. Regular resource attributes that persist to state
# 2. Non-ephemeral outputs
# 3. Data sources that store results in state

# This would fail:
# resource "local_file" "config" {
#   content = local.ephemeral_token  # Can't write ephemeral to state-stored resource
# }

# Ephemeral locals CAN be used in:
# 1. Provider configurations
# 2. Provisioner connection blocks and inline commands
# 3. Ephemeral resource arguments
# 4. Ephemeral output values
```

## Conclusion

Ephemeral locals extend the ephemeral ecosystem by allowing you to compute derived values from ephemeral sources without persisting them to state. This is essential for building authentication flows, credential pipelines, and temporary configuration computation that should remain truly transient. By understanding ephemeral propagation - where accessing an ephemeral source makes the derived value ephemeral - you can build secure, zero-persistence secret handling pipelines in OpenTofu.
