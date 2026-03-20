# How to Use Ephemeral Values in Connection Blocks in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ephemeral Resources, Connection Blocks, SSH, WinRM, HCL, Infrastructure as Code

Description: Learn how to use ephemeral values in connection blocks in OpenTofu to securely pass SSH keys and passwords without storing them in state.

---

Connection blocks configure how OpenTofu connects to a remote machine for provisioner execution. They support ephemeral values, making them a safe place to supply SSH private keys, passwords, and certificates that should not be stored in state.

---

## SSH Connection with Ephemeral Private Key

```hcl
# Fetch SSH private key from Secrets Manager - never stored in state

ephemeral "aws_secretsmanager_secret_version" "ssh_private_key" {
  secret_id = "production/ssh/ec2-deploy-key"
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  key_name      = aws_key_pair.deploy.key_name

  connection {
    type        = "ssh"
    user        = "ec2-user"
    host        = self.public_ip

    # Ephemeral value: not stored in state
    private_key = ephemeral.aws_secretsmanager_secret_version.ssh_private_key.secret_string
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo systemctl restart app",
    ]
  }
}
```

---

## WinRM Connection with Ephemeral Password

```hcl
# Fetch Windows administrator password ephemerally
ephemeral "aws_secretsmanager_secret_version" "win_password" {
  secret_id = "production/windows/admin-password"
}

resource "aws_instance" "windows" {
  ami           = data.aws_ami.windows.id
  instance_type = "t3.medium"

  connection {
    type     = "winrm"
    user     = "Administrator"
    host     = self.public_ip
    https    = true
    insecure = false

    # Ephemeral password
    password = ephemeral.aws_secretsmanager_secret_version.win_password.secret_string
  }

  provisioner "remote-exec" {
    inline = [
      "powershell -Command Install-WindowsFeature -name Web-Server",
    ]
  }
}
```

---

## SSH with a Bastion Host

```hcl
ephemeral "aws_secretsmanager_secret_version" "bastion_key" {
  secret_id = "production/ssh/bastion-key"
}

ephemeral "aws_secretsmanager_secret_version" "app_key" {
  secret_id = "production/ssh/app-server-key"
}

resource "aws_instance" "private_app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.private.id

  # Connect through bastion using ephemeral keys for both hops
  connection {
    type        = "ssh"
    user        = "ec2-user"
    host        = self.private_ip
    private_key = ephemeral.aws_secretsmanager_secret_version.app_key.secret_string

    bastion_host        = aws_instance.bastion.public_ip
    bastion_user        = "ec2-user"
    bastion_private_key = ephemeral.aws_secretsmanager_secret_version.bastion_key.secret_string
  }

  provisioner "remote-exec" {
    inline = ["sudo systemctl start app"]
  }
}
```

---

## Certificate-Based Authentication

```hcl
ephemeral "aws_secretsmanager_secret_version" "client_cert" {
  secret_id = "production/tls/client-certificate"
}

resource "null_resource" "tls_configure" {
  connection {
    type        = "ssh"
    user        = "admin"
    host        = var.target_host

    # Certificate and key - neither stored in state
    certificate = ephemeral.aws_secretsmanager_secret_version.client_cert.secret_string
    private_key = ephemeral.tls_private_key.client.private_key_pem
  }

  provisioner "remote-exec" {
    script = "${path.module}/scripts/configure.sh"
  }
}
```

---

## Connection Block in null_resource

Connection blocks on `null_resource` (or `terraform_data`) also support ephemeral values:

```hcl
ephemeral "aws_ssm_parameter" "ssh_password" {
  name            = "/production/servers/ssh-password"
  with_decryption = true
}

resource "null_resource" "post_deploy" {
  triggers = {
    deploy_id = var.deploy_id
  }

  connection {
    type     = "ssh"
    user     = "ubuntu"
    host     = var.server_host
    password = ephemeral.aws_ssm_parameter.ssh_password.value
    # Password never written to state
  }

  provisioner "remote-exec" {
    inline = [
      "cd /app && git pull",
      "sudo systemctl restart app",
    ]
  }
}
```

---

## What Gets Stored in State

When you use ephemeral values in connection blocks:

| Value | Stored in State? |
|---|---|
| `host` | Yes |
| `user` | Yes |
| `private_key` (ephemeral) | No |
| `password` (ephemeral) | No |
| `certificate` (ephemeral) | No |
| `bastion_private_key` (ephemeral) | No |

The connection block's non-secret attributes (host, user, type) are stored, but ephemeral values are not.

---

## Summary

Connection blocks support ephemeral values for `private_key`, `password`, `certificate`, and `bastion_private_key`. This allows configuring SSH and WinRM connections with credentials fetched from Secrets Manager, Parameter Store, or Vault without storing those credentials in the state file. Use ephemeral values in connection blocks whenever connecting to remote machines requires secrets.
