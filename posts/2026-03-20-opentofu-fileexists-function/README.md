# How to Use the fileexists Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the fileexists function in OpenTofu to conditionally read files only when they exist for optional configuration file handling.

## Introduction

The `fileexists` function in OpenTofu returns `true` if a file exists at the given path, and `false` otherwise. It enables conditional file reading - you can check whether an optional override file exists before attempting to read it, preventing errors.

## Syntax

```hcl
fileexists(path)
```

- Returns `true` if the file exists and is readable
- Returns `false` if the file does not exist
- Raises an error for other filesystem errors (e.g., permissions)

## Basic Examples

```hcl
output "file_present" {
  value = fileexists("${path.module}/config/override.json")  # true or false
}

output "conditional_read" {
  value = fileexists("${path.module}/custom.tf") ? "exists" : "not found"
}
```

## Practical Use Cases

### Optional Override Files

```hcl
locals {
  default_config_path  = "${path.module}/config/defaults.json"
  override_config_path = "${path.module}/config/override.json"

  # Use override if it exists, otherwise use defaults
  config = jsondecode(
    fileexists(local.override_config_path)
    ? file(local.override_config_path)
    : file(local.default_config_path)
  )
}
```

### Environment-Specific Configuration

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

locals {
  env_config_path     = "${path.module}/config/${var.environment}.json"
  default_config_path = "${path.module}/config/default.json"

  # Load env-specific config if it exists
  raw_config = fileexists(local.env_config_path)
    ? file(local.env_config_path)
    : file(local.default_config_path)

  config = jsondecode(local.raw_config)
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = lookup(local.config, "instance_type", "t3.small")

  tags = {
    Name = "app-server"
  }
}
```

### Conditional Key File Usage

```hcl
variable "ssh_key_path" {
  type    = string
  default = "~/.ssh/deploy.pub"
}

locals {
  expanded_path = pathexpand(var.ssh_key_path)
  ssh_key       = fileexists(local.expanded_path) ? file(local.expanded_path) : null
}

resource "aws_key_pair" "deploy" {
  count      = local.ssh_key != null ? 1 : 0
  key_name   = "deploy-key"
  public_key = local.ssh_key
}
```

### Optional TLS Certificate Loading

```hcl
locals {
  cert_path = "${path.module}/certs/server.crt"
  key_path  = "${path.module}/certs/server.key"

  use_custom_tls = fileexists(local.cert_path) && fileexists(local.key_path)

  tls_cert = local.use_custom_tls ? file(local.cert_path) : null
  tls_key  = local.use_custom_tls ? file(local.key_path) : null
}
```

## Step-by-Step Usage

1. Determine the optional file path.
2. Call `fileexists(path)` to test.
3. Use ternary expression to conditionally read the file.

```bash
tofu console

> fileexists("./main.tf")
true
> fileexists("./nonexistent.txt")
false
```

## fileexists with pathexpand

```hcl
locals {
  # pathexpand resolves ~ before fileexists checks
  key_path = pathexpand("~/.ssh/id_rsa.pub")
  key_exists = fileexists(local.key_path)
}
```

## Conclusion

The `fileexists` function is the defensive guard for optional file operations in OpenTofu. Use it to check whether environment-specific config files, SSH keys, TLS certificates, or override files exist before attempting to read them. This prevents plan-time errors and enables flexible, environment-adaptive configurations.
