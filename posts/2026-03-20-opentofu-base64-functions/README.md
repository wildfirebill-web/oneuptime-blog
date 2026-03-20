# How to Use the base64decode and base64encode Functions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the base64decode and base64encode functions in OpenTofu to encode and decode binary data and strings for user-data scripts and secret management.

## Introduction

The `base64encode` and `base64decode` functions in OpenTofu convert strings to and from Base64 encoding. They are essential for encoding user-data scripts, secrets, binary content, and any data that must be passed in a Base64-encoded format.

## Syntax

```hcl
base64encode(string)
base64decode(string)
```

- `base64encode` converts a UTF-8 string to its Base64 representation
- `base64decode` converts a Base64 string back to UTF-8

## Basic Examples

```hcl
output "encoded" {
  value = base64encode("hello world")  # Returns "aGVsbG8gd29ybGQ="
}

output "decoded" {
  value = base64decode("aGVsbG8gd29ybGQ=")  # Returns "hello world"
}
```

## Practical Use Cases

### Encoding EC2 User Data

```hcl
locals {
  user_data_script = <<-SCRIPT
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    systemctl start nginx
    systemctl enable nginx
    echo "Server: ${var.server_name}" > /var/www/html/index.html
  SCRIPT

  # EC2 user_data must be base64 encoded
  user_data_b64 = base64encode(local.user_data_script)
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  user_data     = local.user_data_b64

  tags = {
    Name = "web-server"
  }
}
```

### Encoding Kubernetes Secrets

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

resource "kubernetes_secret" "db" {
  metadata {
    name      = "db-credentials"
    namespace = "default"
  }

  data = {
    # Kubernetes secrets are base64-encoded
    password = base64encode(var.db_password)
    username = base64encode("appuser")
  }
}
```

### Decoding SSM Parameter Values

```hcl
data "aws_ssm_parameter" "encoded_config" {
  name = "/app/encoded-config"
}

locals {
  # Decode the base64-encoded config stored in SSM
  config = jsondecode(base64decode(data.aws_ssm_parameter.encoded_config.value))
}
```

### Encoding Cloud Init MIME

```hcl
locals {
  cloud_config = <<-YAML
    #cloud-config
    packages:
      - nginx
      - docker.io
    runcmd:
      - systemctl start nginx
  YAML

  encoded_cloud_config = base64encode(local.cloud_config)
}
```

## Step-by-Step Usage

1. Identify data that needs Base64 encoding/decoding.
2. Apply `base64encode()` for encoding, `base64decode()` for decoding.
3. Test in `tofu console`:

```bash
tofu console

> base64encode("hello")
"aGVsbG8="
> base64decode("aGVsbG8=")
"hello"
```

## base64encode vs filebase64

| Function | Input | Use Case |
|----------|-------|----------|
| `base64encode(string)` | String value | Encode string content |
| `filebase64(path)` | File path | Encode file contents |

## Conclusion

The `base64encode` and `base64decode` functions are essential for working with EC2 user data, Kubernetes secrets, and any API that requires Base64-encoded input. They handle the encoding transparently so your configuration stays readable.
