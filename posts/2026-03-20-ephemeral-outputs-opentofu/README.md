# How to Use Ephemeral Outputs in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Outputs, Ephemeral, Security, Infrastructure as Code, DevOps

Description: A guide to using ephemeral outputs in OpenTofu to expose temporary values that are never stored in the state file.

## Introduction

Ephemeral outputs in OpenTofu (introduced in 1.10) are output values that exist only during the plan/apply execution and are never written to the state file. Unlike sensitive outputs (which are stored in state but masked in display), ephemeral outputs are truly transient — they cannot be persisted.

## Declaring Ephemeral Outputs

```hcl
# outputs.tf

output "temporary_token" {
  description = "Temporary authentication token (not stored in state)"
  value       = ephemeral.vault_secret.app_token.data.token
  ephemeral   = true  # Never written to state
}

output "session_credentials" {
  description = "Temporary AWS session credentials (not stored in state)"
  value = {
    access_key    = ephemeral.aws_temporary_credentials.main.access_key
    secret_key    = ephemeral.aws_temporary_credentials.main.secret_key
    session_token = ephemeral.aws_temporary_credentials.main.session_token
  }
  ephemeral = true
  sensitive = true  # Also mask from display
}
```

## Ephemeral Outputs vs Sensitive Outputs

```hcl
# Sensitive output: stored in state (encrypted ideally), masked in display
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
  # ^ In state file, but masked in terminal output
}

# Ephemeral output: NEVER stored in state
output "vault_token" {
  value     = data.vault_generic_secret.app.data.token
  ephemeral = true
  sensitive = true  # Also mask from display
  # ^ NOT in state file, only exists during execution
}
```

## Using Ephemeral Outputs Between Modules

```hcl
# When an output is ephemeral, it can only be used by resources
# that also support ephemeral values (providers, provisioners, etc.)

# modules/vault-auth/outputs.tf
output "access_token" {
  value     = vault_approle_auth_login.main.client_token
  ephemeral = true
  sensitive = true
}
```

```hcl
# root/main.tf - Using ephemeral module output
module "vault_auth" {
  source = "./modules/vault-auth"
}

# Use ephemeral output in a provider configuration
provider "vault" {
  token = module.vault_auth.access_token  # Ephemeral value
}
```

## Limitations of Ephemeral Outputs

```hcl
# Ephemeral outputs CANNOT be used in:
# 1. Regular resource attributes stored in state
# 2. Data source attributes that are stored in state
# 3. Non-ephemeral local values
# 4. Non-ephemeral output values

# This would fail:
# resource "local_file" "token" {
#   content = output.temporary_token  # Can't use ephemeral in state-stored resource
# }

# Ephemeral values can be used in:
# - Provider configurations
# - Provisioner connection blocks
# - Provisioner inline commands
# - Ephemeral resources
# - Ephemeral locals
```

## Ephemeral Output Use Cases

```hcl
# Use case 1: Dynamic provider credentials
output "aws_temp_credentials" {
  value = {
    access_key = vault_aws_access_credentials.main.access_key
    secret_key = vault_aws_access_credentials.main.secret_key
    token      = vault_aws_access_credentials.main.security_token
  }
  ephemeral = true
  sensitive = true
}

# Use case 2: One-time setup tokens
output "setup_token" {
  value     = random_password.setup.result
  ephemeral = true
  # Used once for initial setup, should not be persisted
}

# Use case 3: Short-lived certificates
output "client_certificate" {
  value = {
    cert    = tls_locally_signed_cert.client.cert_pem
    key     = tls_private_key.client.private_key_pem
    ca_cert = tls_self_signed_cert.ca.cert_pem
  }
  ephemeral = true
  sensitive = true
}
```

## Conclusion

Ephemeral outputs provide the strongest security guarantee for truly transient values — they are provably not stored in state. This is essential for temporary credentials, one-time tokens, and other values that should exist only for the duration of a deployment. Use ephemeral outputs for any value that would be a security risk if persisted, and combine them with ephemeral variables and resources for a comprehensive ephemeral secret management strategy.
