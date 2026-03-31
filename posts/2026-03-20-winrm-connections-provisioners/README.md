# How to Configure WinRM Connections in OpenTofu Provisioners

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, WinRM, Window, Provisioners, Infrastructure as Code

Description: Learn how to configure WinRM connection blocks in OpenTofu provisioners to remotely execute scripts and commands on Windows instances after they are created.

## Introduction

WinRM (Windows Remote Management) is the Windows equivalent of SSH for remote command execution. OpenTofu supports WinRM connections in provisioners, allowing you to configure Windows instances at creation time using `remote-exec` and `file` provisioners.

## Prerequisites

- A Windows instance with WinRM enabled
- Firewall rules allowing WinRM (port 5985 for HTTP, 5986 for HTTPS)
- Valid Windows credentials

## Enabling WinRM on Windows

Include this user data script when creating Windows EC2 instances:

```hcl
resource "aws_instance" "windows" {
  ami           = data.aws_ami.windows.id
  instance_type = "t3.medium"

  user_data = <<-EOF
    <powershell>
    # Enable WinRM
    Enable-PSRemoting -Force
    Set-Item -Path WSMan:\localhost\Service\Auth\Basic -Value $true
    Set-Item -Path WSMan:\localhost\Service\AllowUnencrypted -Value $true
    New-NetFirewallRule -DisplayName "WinRM HTTP" -Direction Inbound `
      -Protocol TCP -LocalPort 5985 -Action Allow
    </powershell>
  EOF
}
```

## WinRM Connection Block

```hcl
resource "aws_instance" "windows_app" {
  ami           = data.aws_ami.windows.id
  instance_type = "t3.medium"
  key_name      = aws_key_pair.windows.key_name

  connection {
    type     = "winrm"
    user     = "Administrator"
    password = var.admin_password
    host     = self.public_ip
    port     = 5985
    https    = false
    insecure = true
    timeout  = "10m"
  }

  provisioner "remote-exec" {
    inline = [
      "powershell Install-WindowsFeature -Name Web-Server -IncludeManagementTools",
      "powershell Start-Service W3SVC"
    ]
  }
}
```

## HTTPS WinRM Connection

For production, use HTTPS:

```hcl
connection {
  type     = "winrm"
  user     = "Administrator"
  password = var.admin_password
  host     = self.public_ip
  port     = 5986
  https    = true
  insecure = false   # Set to true to skip cert validation (dev only)
  timeout  = "10m"
}
```

## Running PowerShell Scripts

```hcl
provisioner "file" {
  source      = "scripts/configure.ps1"
  destination = "C:\\Temp\\configure.ps1"
}

provisioner "remote-exec" {
  inline = [
    "powershell -ExecutionPolicy Bypass -File C:\\Temp\\configure.ps1"
  ]
}
```

## Security Group for WinRM

```hcl
resource "aws_security_group_rule" "winrm" {
  type              = "ingress"
  from_port         = 5985
  to_port           = 5986
  protocol          = "tcp"
  cidr_blocks       = [var.management_cidr]
  security_group_id = aws_security_group.windows.id
}
```

## Best Practices

- Use HTTPS (port 5986) in production environments
- Restrict WinRM access to management CIDRs only
- Consider AWS Systems Manager Session Manager as a more secure alternative
- Use strong, unique passwords and rotate them regularly

## Conclusion

WinRM connection blocks in OpenTofu provisioners enable Windows instance configuration during infrastructure creation. For production workloads, use HTTPS WinRM connections and restrict access to management networks, or consider AWS SSM as a more secure alternative that doesn't require open WinRM ports.
