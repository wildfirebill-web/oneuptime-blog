# How to Use the sha256 and sha512 Functions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the sha256 and sha512 functions in OpenTofu to compute cryptographic hashes for Lambda deployment verification and content integrity checks.

## Introduction

The `sha256` and `sha512` functions in OpenTofu compute SHA-256 and SHA-512 hashes of strings respectively. These are cryptographically stronger than `md5` and `sha1`, making them suitable for security-sensitive contexts like Lambda deployment verification and content integrity validation.

## Syntax

```hcl
sha256(string)
sha512(string)
```

- `sha256` returns a 64-character hex string
- `sha512` returns a 128-character hex string

## Basic Examples

```hcl
output "sha256_hash" {
  value = sha256("hello world")
  # Returns 64-char hex string
}

output "sha512_hash" {
  value = sha512("hello world")
  # Returns 128-char hex string
}
```

## Practical Use Cases

### Content Integrity Verification

```hcl
variable "expected_sha256" {
  type        = string
  description = "Expected SHA256 of the config file"
}

locals {
  config_content = file("${path.module}/config/app.json")
  actual_sha256  = sha256(local.config_content)
}

resource "null_resource" "integrity_check" {
  lifecycle {
    precondition {
      condition     = local.actual_sha256 == var.expected_sha256
      error_message = "Config file integrity check failed. Expected ${var.expected_sha256}, got ${local.actual_sha256}."
    }
  }
}
```

### Lambda Source Code Hash

```hcl
locals {
  # Track source changes for deployment
  source_code_hash = sha256(join("", [
    for f in fileset("${path.module}/lambda/src", "**/*.js") :
    sha256(file("${path.module}/lambda/src/${f}"))
  ]))
}

resource "null_resource" "lambda_deploy" {
  triggers = {
    source_hash = local.source_code_hash
  }

  provisioner "local-exec" {
    command = "npm run deploy"
    working_dir = "${path.module}/lambda"
  }
}
```

### Generating Secure Token Hashes

```hcl
variable "api_secret" {
  type      = string
  sensitive = true
}

locals {
  # Store hash instead of plaintext secret
  api_secret_hash = sha256(var.api_secret)
}

resource "aws_ssm_parameter" "api_secret_hash" {
  name  = "/app/api-secret-hash"
  type  = "String"
  value = local.api_secret_hash
}
```

### HMAC-Like Configuration Signing

```hcl
variable "config_salt" {
  type    = string
  default = "my-company-salt-2024"
}

locals {
  config_data      = jsonencode(var.application_config)
  signed_config    = sha256("${var.config_salt}${local.config_data}")
}

output "config_signature" {
  value = local.signed_config
}
```

## Step-by-Step Usage

```bash
tofu console

> sha256("hello")
"2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"
> length(sha256("x"))
64
> length(sha512("x"))
128
```

## Choosing Between sha256 and sha512

| Scenario | Recommendation |
|----------|---------------|
| Lambda `source_code_hash` | Use `base64sha256` (AWS requirement) |
| Content integrity | `sha256` (faster, sufficient security) |
| High-security signatures | `sha512` |
| Unique names | `sha256` (shorter than sha512) |

## Conclusion

The `sha256` and `sha512` functions provide cryptographically strong hashing in OpenTofu. Use `sha256` as the default secure hash, `sha512` when stronger guarantees are needed, and `base64sha256` specifically for AWS Lambda source code hash verification.
