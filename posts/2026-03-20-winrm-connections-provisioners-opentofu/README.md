# How to Configure WinRM Connections for Provisioners in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Provisioners, WinRM, Window, Infrastructure as Code

Description: Learn how to configure WinRM connection blocks in OpenTofu to authenticate provisioners to Windows remote resources for file transfers and command execution.

## Introduction

Windows instances do not use SSH by default-they use WinRM (Windows Remote Management). OpenTofu supports WinRM connections in `connection` blocks, enabling `file` and `remote-exec` provisioners to run against Windows servers, IIS hosts, and Active Directory domain controllers.

## Enabling WinRM on an EC2 Windows Instance

The easiest way is to include a user data script that enables and configures WinRM:

```hcl
resource "aws_instance" "windows" {
  ami           = data.aws_ami.windows_2022.id
  instance_type = "t3.medium"
  key_name      = var.key_name

  # User data script that enables WinRM on first boot
  user_data = <<-EOF
    <powershell>
    # Enable WinRM with basic authentication (HTTPS recommended for production)
    winrm quickconfig -quiet
    winrm set winrm/config/service '@{AllowUnencrypted="true"}'
    winrm set winrm/config/service/auth '@{Basic="true"}'

    # Set the administrator password
    net user Administrator "${var.admin_password}"
    </powershell>
  EOF

  # Allow WinRM traffic on port 5985
  vpc_security_group_ids = [aws_security_group.winrm.id]

  connection {
    type     = "winrm"
    user     = "Administrator"
    password = var.admin_password
    host     = self.public_ip

    # Wait for WinRM to become available after boot
    timeout  = "10m"
  }

  provisioner "remote-exec" {
    inline = [
      "echo 'Windows instance is ready'",
      "dir C:\\",
    ]
  }
}
```

## WinRM Connection Block Arguments

```hcl
connection {
  type     = "winrm"    # Required: must be "winrm" for Windows
  user     = "Administrator"
  password = var.admin_password
  host     = self.public_ip

  # Optional: use HTTPS (port 5986) instead of HTTP (5985)
  https    = true
  port     = 5986

  # Skip certificate verification for self-signed certs
  insecure = true

  # Use NTLM authentication instead of Basic (more secure)
  use_ntlm = true

  # Connection timeout
  timeout  = "10m"
}
```

## Security Group for WinRM

Always create an explicit security group allowing WinRM access:

```hcl
resource "aws_security_group" "winrm" {
  name        = "allow-winrm"
  description = "Allow WinRM access for provisioners"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "WinRM HTTP"
    from_port   = 5985
    to_port     = 5985
    protocol    = "tcp"
    # Restrict to your deployment IP or bastion range
    cidr_blocks = [var.allowed_cidr]
  }

  ingress {
    description = "WinRM HTTPS"
    from_port   = 5986
    to_port     = 5986
    protocol    = "tcp"
    cidr_blocks = [var.allowed_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## Uploading Files and Running PowerShell Scripts

```hcl
resource "aws_instance" "iis_server" {
  ami           = data.aws_ami.windows_2022.id
  instance_type = "t3.medium"

  connection {
    type     = "winrm"
    user     = "Administrator"
    password = var.admin_password
    host     = self.public_ip
    https    = false
    timeout  = "15m"
  }

  # Upload a PowerShell setup script
  provisioner "file" {
    source      = "${path.module}/scripts/install-iis.ps1"
    destination = "C:\\Windows\\Temp\\install-iis.ps1"
  }

  # Execute the PowerShell script
  provisioner "remote-exec" {
    inline = [
      "powershell.exe -ExecutionPolicy Bypass -File C:\\Windows\\Temp\\install-iis.ps1",
    ]
  }
}
```

## Best Practices for WinRM

**Use HTTPS in production.** HTTP WinRM (port 5985) transmits credentials and commands in plaintext. Configure a certificate and use port 5986 for production workloads.

**Prefer EC2 Run Command.** AWS Systems Manager Run Command can execute scripts on Windows instances without opening WinRM ports. This is more secure than exposing WinRM to the internet.

**Consider cloud-init / EC2 user data.** For most Windows setup tasks, a PowerShell user data script is simpler and more reliable than WinRM provisioners.

## Conclusion

WinRM connections bring OpenTofu provisioners to Windows workloads. While functional, they expose network ports and require careful security configuration. For new Windows deployments, evaluate whether EC2 Run Command or user data scripts can accomplish the same goal with a smaller attack surface.
