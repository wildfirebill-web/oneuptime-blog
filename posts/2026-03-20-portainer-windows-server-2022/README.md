# How to Install Portainer on Windows Server 2022 with Docker - 2022

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Windows Server, Docker, Windows Containers, Self-Hosted, Enterprise

Description: Install Docker Engine and Portainer on Windows Server 2022 to manage both Linux containers (via WSL2) and native Windows containers from a single interface.

## Introduction

Windows Server 2022 supports Docker in two modes: Linux containers via WSL2 integration, and native Windows containers. Portainer runs on both and provides a unified management interface. This guide covers setting up Docker and Portainer on Windows Server 2022.

## Prerequisites

- Windows Server 2022 (Standard or Datacenter)
- Administrator access
- Internet connectivity
- At least 8GB RAM and 2 vCPUs

## Step 1: Install Required Windows Features

Open PowerShell as Administrator:

```powershell
# Install Hyper-V and containers features

Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart
Install-WindowsFeature -Name Containers

# Or in one command
Install-WindowsFeature -Name Hyper-V, Containers -IncludeManagementTools -Restart
```

## Step 2: Install Docker on Windows Server 2022

### Option A: Docker Engine (Community)

```powershell
# Install Docker Engine
Invoke-WebRequest -UseBasicParsing "https://raw.githubusercontent.com/microsoft/Windows-Containers/Main/helpful_tools/Install-DockerCE/install-docker-ce.ps1" -o install-docker-ce.ps1
.\install-docker-ce.ps1
```

### Option B: Docker Desktop (for GUI)

Download Docker Desktop for Windows from docker.com and install it.

### Option C: Manual Installation

```powershell
# Download and install Docker Engine
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

# Install NuGet provider
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force

# Install Docker module
Install-Module -Name Docker -Force

# Install Docker
Install-Package -Name docker -ProviderName DockerMsftProvider -Force
Start-Service docker
Set-Service docker -StartupType Automatic
```

## Step 3: Enable WSL2 for Linux Containers

```powershell
# Enable WSL
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Restart
Restart-Computer

# After restart, set WSL2 as default
wsl --set-default-version 2

# Install Ubuntu (for Linux containers)
wsl --install -d Ubuntu-22.04
```

## Step 4: Switch Docker to Linux Containers Mode

```powershell
# Switch to Linux containers (via WSL2)
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# Verify
docker version
docker info | Select-String "OS/Arch"
```

## Step 5: Deploy Portainer

```powershell
# Create Portainer data volume
docker volume create portainer_data

# Deploy Portainer (Linux container mode)
docker run -d `
  --name portainer `
  --restart=unless-stopped `
  -p 9000:9000 `
  -p 9443:9443 `
  -v //./pipe/docker_engine://./pipe/docker_engine `
  -v portainer_data:C:/data `
  portainer/portainer-ce:latest
```

Note the Windows-specific Docker socket path: `//./pipe/docker_engine`.

## Step 6: Configure Windows Firewall

```powershell
# Allow Portainer ports through Windows Firewall
New-NetFirewallRule -DisplayName "Portainer HTTP" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 9000 `
  -Action Allow

New-NetFirewallRule -DisplayName "Portainer HTTPS" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 9443 `
  -Action Allow
```

## Step 7: Access Portainer

Navigate to `http://localhost:9000` or `http://<server-ip>:9000`.

## Step 8: Managing Windows Containers with Portainer

Switch Docker to Windows container mode:

```powershell
# Switch to Windows containers
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# Deploy a Windows container via Portainer
# In Portainer, create a new container:
# Image: mcr.microsoft.com/windows/servercore:ltsc2022
# (Windows container images are larger - typically 3-5GB)
```

## Troubleshooting

### Docker Service Won't Start

```powershell
# Check Windows Event Log
Get-EventLog -LogName Application -Source "*Docker*" -Newest 20

# Check Docker daemon logs
Get-Content C:\ProgramData\Docker\config\daemon.json
```

### Port Already in Use

```powershell
# Find what's using port 9000
netstat -ano | findstr :9000
Get-Process -Id <PID>
```

## Conclusion

Portainer on Windows Server 2022 provides unified container management for both Linux and Windows containers. While the setup is more involved than on Linux, it enables enterprise scenarios where Windows Server workloads need to coexist with Linux containers. The WSL2 backend provides excellent Linux container performance even on Windows.
