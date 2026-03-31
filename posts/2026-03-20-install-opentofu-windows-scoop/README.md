# How to Install OpenTofu on Windows Using Scoop

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Window, Scoop, Installation, Infrastructure as Code, DevOps

Description: A guide to installing OpenTofu on Windows using the Scoop package manager without requiring administrator privileges.

## Introduction

Scoop is a command-line package manager for Windows that installs programs in your user directory, eliminating the need for admin privileges in most cases. This makes it ideal for developer workstations where you may not have full administrative access.

## Prerequisites

- Windows 10 or Windows 11
- PowerShell 5.1 or later
- Internet access

## Step 1: Install Scoop

```powershell
# Set execution policy for current user (no admin needed)

Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# Install Scoop
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression

# Verify installation
scoop --version
```

## Step 2: Add the Main Bucket

```powershell
# Add the main bucket which contains opentofu
scoop bucket add main

# Verify buckets
scoop bucket list
```

## Step 3: Install OpenTofu

```powershell
# Install OpenTofu
scoop install opentofu

# Or install a specific version
scoop install opentofu@1.9.0
```

## Step 4: Verify the Installation

```powershell
# Check installed version
tofu version

# Output:
# OpenTofu v1.9.0
# on windows_amd64

# Check binary location
where tofu
# C:\Users\YourUsername\scoop\apps\opentofu\current\tofu.exe
```

## Advantages of Scoop Over Chocolatey

- Does **not** require administrator privileges
- Installs to user home directory (`~/scoop/apps/`)
- Easy version management with `scoop reset`
- Clean uninstalls with no registry changes

## Managing Multiple Versions with Scoop

```powershell
# Install a specific version alongside current
scoop install opentofu@1.8.0

# Switch between installed versions
scoop reset opentofu@1.8.0  # Switch to 1.8.0
scoop reset opentofu@1.9.0  # Switch back to 1.9.0

# Check which version is active
tofu version
```

## Quick Start Project

```powershell
# Create a test project
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\projects\tofu-demo"
Set-Location "$env:USERPROFILE\projects\tofu-demo"
```

```hcl
# main.tf
terraform {
  required_version = ">= 1.6"
}

locals {
  user = "Windows User"
}

output "greeting" {
  value = "Hello, ${local.user}! OpenTofu via Scoop is working!"
}
```

```powershell
tofu init
tofu apply -auto-approve
```

## Working with AWS on Windows

```powershell
# Set AWS credentials (prefer IAM roles in production)
$env:AWS_ACCESS_KEY_ID     = "AKIAIOSFODNN7EXAMPLE"
$env:AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
$env:AWS_DEFAULT_REGION    = "us-east-1"

# Optionally store in a profile file
# ~/.aws/credentials is read by the AWS provider
```

## Updating OpenTofu via Scoop

```powershell
# Update Scoop itself
scoop update

# Upgrade OpenTofu
scoop update opentofu

# Verify
tofu version
```

## Removing OpenTofu

```powershell
# Uninstall OpenTofu
scoop uninstall opentofu

# Optionally, remove the bucket
scoop bucket rm main
```

## Comparison: Scoop vs Chocolatey vs Winget

| Feature | Scoop | Chocolatey | Winget |
|---------|-------|------------|--------|
| Admin required | No | Sometimes | No |
| Install location | User dir | System dir | Both |
| Version pinning | Easy | Supported | Limited |
| CLI focused | Yes | Both | Both |

## Conclusion

Scoop is an excellent choice for installing OpenTofu on Windows, particularly in corporate environments where admin rights are restricted. The user-level installation ensures no system-wide changes, while the simple CLI makes version management straightforward. With OpenTofu installed via Scoop, you can start managing infrastructure as code directly from PowerShell or Command Prompt.
