# How to Manage SSH Keys with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, SSH Keys, AWS, EC2, Security, Infrastructure as Code

Description: Learn how to manage SSH key pairs with OpenTofu — generating keys with the TLS provider, importing existing public keys, storing private keys in AWS Secrets Manager, and rotating keys safely.

## Introduction

SSH key pairs authenticate access to EC2 instances. OpenTofu manages key pairs declaratively — generating new RSA/ECDSA keys, importing existing public keys into AWS, and securely storing private key material in Secrets Manager. Using code-managed keys ensures consistent key distribution across environments and enables key rotation without manual steps.

## Generate Key Pair with TLS Provider

```hcl
# Generate RSA private key locally
resource "tls_private_key" "app" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

# Register the public key with AWS
resource "aws_key_pair" "app" {
  key_name   = "${var.environment}-app-key"
  public_key = tls_private_key.app.public_key_openssh

  tags = { Environment = var.environment }
}

# Store the private key in Secrets Manager
resource "aws_secretsmanager_secret" "app_private_key" {
  name        = "/${var.environment}/ec2/app-private-key"
  description = "Private SSH key for ${var.environment} app instances"

  tags = { Environment = var.environment }
}

resource "aws_secretsmanager_secret_version" "app_private_key" {
  secret_id     = aws_secretsmanager_secret.app_private_key.id
  secret_string = tls_private_key.app.private_key_pem
}
```

## Import Existing Public Key

```hcl
# Import a public key that was generated outside OpenTofu
# (e.g., ssh-keygen -t ed25519 -C "ops@company.com")
resource "aws_key_pair" "ops" {
  key_name   = "${var.environment}-ops-key"
  public_key = file("${path.module}/keys/${var.environment}/ops.pub")

  tags = { Environment = var.environment }
}
```

## EC2 Instance Referencing Key Pair

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.private[0].id
  key_name      = aws_key_pair.app.key_name

  vpc_security_group_ids = [aws_security_group.app.id]

  tags = { Name = "${var.environment}-app" }
}
```

## ECDSA Key (More Efficient than RSA)

```hcl
resource "tls_private_key" "app_ecdsa" {
  algorithm   = "ECDSA"
  ecdsa_curve = "P256"
}

resource "aws_key_pair" "app_ecdsa" {
  key_name   = "${var.environment}-app-ecdsa-key"
  public_key = tls_private_key.app_ecdsa.public_key_openssh
}
```

## Key Rotation Strategy

```hcl
# Use a version suffix to support rotation without downtime
variable "key_version" {
  description = "Increment to rotate SSH keys"
  default     = "v1"
}

resource "tls_private_key" "app_versioned" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "app_versioned" {
  key_name   = "${var.environment}-app-key-${var.key_version}"
  public_key = tls_private_key.app_versioned.public_key_openssh
}

# Reference the versioned key in the launch template
resource "aws_launch_template" "app" {
  name          = "${var.environment}-app-lt"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  key_name      = aws_key_pair.app_versioned.key_name
}
```

## Retrieve Private Key from Secrets Manager (for CI/CD)

```hcl
# Data source to retrieve the key in another module or pipeline
data "aws_secretsmanager_secret_version" "app_private_key" {
  secret_id = "/${var.environment}/ec2/app-private-key"
}

# Use in a null_resource for one-time setup tasks
resource "null_resource" "setup" {
  triggers = {
    instance_id = aws_instance.app.id
  }

  connection {
    type        = "ssh"
    host        = aws_instance.app.private_ip
    user        = "ec2-user"
    private_key = data.aws_secretsmanager_secret_version.app_private_key.secret_string

    bastion_host = aws_eip.bastion.public_ip
    bastion_user = "ec2-user"
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "echo 'Setup complete'"
    ]
  }
}
```

## Outputs (Safe — Public Key Only)

```hcl
output "key_pair_name" {
  value       = aws_key_pair.app.key_name
  description = "AWS key pair name for reference in other modules"
}

output "public_key_openssh" {
  value       = tls_private_key.app.public_key_openssh
  description = "Public key in OpenSSH format — safe to share"
}

# Never output private keys directly — use Secrets Manager ARN
output "private_key_secret_arn" {
  value       = aws_secretsmanager_secret.app_private_key.arn
  description = "Secrets Manager ARN for the private key"
}
```

## Conclusion

Managing SSH keys with OpenTofu's TLS provider gives you code-managed key generation and rotation. Store private key material exclusively in Secrets Manager — never in OpenTofu state files (state is stored encrypted in S3 with state encryption enabled). Prefer ECDSA P256 over RSA for better performance with equivalent security. Rotate keys by incrementing a `key_version` variable, which creates a new key pair and triggers an instance refresh in the Auto Scaling Group to deploy instances with the new key.
