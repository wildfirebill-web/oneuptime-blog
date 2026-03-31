# How to Use Ephemeral Values in Connection Blocks in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ephemeral Values, Connection Blocks, Provisioner, SSH, Infrastructure as Code, DevOps

Description: A guide to using ephemeral values in connection blocks in OpenTofu to securely manage SSH keys and passwords without persisting them in state.

## Introduction

Connection blocks in OpenTofu define how provisioners connect to remote machines. Using ephemeral values for connection credentials (SSH private keys, passwords, bastion host keys) ensures these sensitive values are used during provisioning but never written to the state file.

## Basic Ephemeral SSH Key in Connection Block

```hcl
# Generate an ephemeral SSH key for provisioning

ephemeral "tls_private_key" "provisioner" {
  algorithm = "ED25519"
}

# Add the public key to the instance
resource "aws_key_pair" "provisioner" {
  key_name   = "temp-provisioner-${var.deployment_id}"
  public_key = ephemeral.tls_private_key.provisioner.public_key_openssh
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  key_name      = aws_key_pair.provisioner.key_name

  connection {
    type        = "ssh"
    user        = "ubuntu"
    # Private key is ephemeral - not stored in state
    private_key = ephemeral.tls_private_key.provisioner.private_key_openssh
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update -y",
      "sudo apt-get install -y nginx",
    ]
  }
}
```

## Using Vault for SSH Keys

```hcl
# Get SSH private key from Vault
ephemeral "vault_kv_secret_v2" "ssh_key" {
  mount = "secret"
  name  = "platform/provisioner-ssh-key"
}

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  key_name      = var.existing_key_name

  connection {
    type        = "ssh"
    user        = "ec2-user"
    # Key fetched from Vault, not stored in state
    private_key = ephemeral.vault_kv_secret_v2.ssh_key.data["private_key"]
    host        = self.private_ip
  }

  provisioner "remote-exec" {
    script = "${path.module}/scripts/configure.sh"
  }
}
```

## WinRM Connection with Ephemeral Password

```hcl
# Get Windows admin password from Secrets Manager
ephemeral "aws_secretsmanager_secret_version" "windows_admin" {
  secret_id = "platform/windows-admin-password"
}

resource "aws_instance" "windows" {
  ami           = var.windows_ami_id
  instance_type = "t3.medium"

  connection {
    type     = "winrm"
    user     = "Administrator"
    # Password ephemeral - not in state
    password = jsondecode(
      ephemeral.aws_secretsmanager_secret_version.windows_admin.secret_string
    ).password
    host     = self.public_ip
    https    = true
    insecure = false
  }

  provisioner "remote-exec" {
    inline = [
      "powershell.exe -Command \"Install-WindowsFeature -Name Web-Server\"",
    ]
  }
}
```

## Bastion Host Connection with Ephemeral Keys

```hcl
# Get both bastion and target host SSH keys from Vault
ephemeral "vault_kv_secret_v2" "bastion_key" {
  mount = "secret"
  name  = "platform/bastion-ssh-key"
}

ephemeral "vault_kv_secret_v2" "target_key" {
  mount = "secret"
  name  = "platform/target-ssh-key"
}

resource "aws_instance" "private_server" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.private.id

  connection {
    type        = "ssh"
    user        = "ubuntu"
    # Target server key
    private_key = ephemeral.vault_kv_secret_v2.target_key.data["private_key"]
    host        = self.private_ip

    # Bastion host configuration
    bastion_host        = aws_instance.bastion.public_ip
    bastion_user        = "ubuntu"
    # Bastion key - also ephemeral
    bastion_private_key = ephemeral.vault_kv_secret_v2.bastion_key.data["private_key"]
  }

  provisioner "remote-exec" {
    inline = [
      "sudo systemctl start myapp",
    ]
  }
}
```

## Dynamic SSH Key Generation per Deployment

```hcl
# Generate unique key per deployment for maximum security
resource "terraform_data" "deployment_id" {
  input = uuid()
}

ephemeral "tls_private_key" "deploy" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

# Temporary key pair - add to instance at creation
resource "aws_key_pair" "deploy" {
  key_name   = "deploy-${terraform_data.deployment_id.output}"
  public_key = ephemeral.tls_private_key.deploy.public_key_openssh

  lifecycle {
    # Clean up key pair after deployment
    create_before_destroy = true
  }
}

resource "aws_instance" "configured" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  key_name      = aws_key_pair.deploy.key_name

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = ephemeral.tls_private_key.deploy.private_key_openssh
    host        = self.public_ip
    timeout     = "5m"
  }

  provisioner "remote-exec" {
    inline = [
      "sudo bash /tmp/setup.sh",
    ]
  }
}
```

## Null Resource with Ephemeral Connection

```hcl
# Run configuration after instance is ready
ephemeral "aws_secretsmanager_secret_version" "instance_key" {
  secret_id = "myapp/instance-ssh-key"
}

resource "terraform_data" "configure_instance" {
  triggers_replace = {
    instance_id  = aws_instance.app.id
    config_hash  = sha256(file("${path.module}/scripts/configure.sh"))
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = jsondecode(
      ephemeral.aws_secretsmanager_secret_version.instance_key.secret_string
    ).private_key
    host        = aws_instance.app.public_ip
  }

  provisioner "file" {
    source      = "${path.module}/scripts/configure.sh"
    destination = "/tmp/configure.sh"
  }

  provisioner "remote-exec" {
    inline = ["bash /tmp/configure.sh"]
  }
}
```

## Conclusion

Using ephemeral values in connection blocks prevents SSH private keys and passwords from being written to the state file - a critical security improvement. Every time OpenTofu runs, it fetches fresh credentials from your secrets management system, uses them for provisioning, and discards them. This approach works well with secrets rotation, as each deployment automatically picks up current credentials. Combine ephemeral SSH keys with short-lived key pairs (deleted after provisioning) for maximum security in production environments.
