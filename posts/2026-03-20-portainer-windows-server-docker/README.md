# How to Install Portainer on Windows Server with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, windows-server, docker, installation, enterprise

Description: A guide to installing Portainer CE on Windows Server with Docker Engine (Linux containers via Hyper-V), covering enterprise deployment scenarios.

## Overview

Windows Server can run Docker containers in two modes: Windows containers (native) and Linux containers (via Hyper-V isolation). Portainer CE runs as a Linux container, so you need Docker Desktop or Docker Engine with Linux container support on Windows Server. This guide covers deploying Portainer on Windows Server 2019 and 2022.

## Prerequisites

- Windows Server 2019 or 2022 (Datacenter or Standard)
- Hyper-V role enabled
- Minimum: 4GB RAM, 40GB disk
- Internet access

## Step 1: Enable Required Windows Features

```powershell
# Enable Hyper-V and Containers features
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All -NoRestart
Enable-WindowsOptionalFeature -Online -FeatureName Containers -All -NoRestart

# Restart to apply changes
Restart-Computer
```

## Step 2: Install Docker on Windows Server

### Option A: Install Docker Desktop (Recommended)

Download Docker Desktop for Windows from https://www.docker.com/products/docker-desktop and install with default settings. Ensure "Use WSL2 based engine" is selected if WSL2 is available.

### Option B: Docker Engine (Server-only, No GUI)

```powershell
# Install Docker Engine via PowerShell
Install-Module -Name DockerMsftProvider -Repository PSGallery -Force
Install-Package -Name docker -ProviderName DockerMsftProvider -Force

# Configure Docker to use Linux containers
# (Requires Docker Desktop for Linux containers on Windows Server without WSL2)
```

## Step 3: Configure Linux Containers Mode

```powershell
# Switch to Linux containers mode (if using Docker Desktop)
# Right-click Docker Desktop tray icon -> "Switch to Linux containers"

# Or via command line:
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchLinuxEngine
```

## Step 4: Deploy Portainer CE

```powershell
# Create data volume
docker volume create portainer_data

# Deploy Portainer CE
docker run -d `
  -p 8000:8000 `
  -p 9443:9443 `
  --name portainer `
  --restart=always `
  -v //./pipe/docker_engine://./pipe/docker_engine `
  -v portainer_data:/data `
  portainer/portainer-ce:latest

# Verify
docker ps
```

Note: Windows uses `//./pipe/docker_engine` for the Docker socket path.

## Step 5: Configure Windows Firewall

```powershell
# Allow Portainer ports through Windows Firewall
New-NetFirewallRule `
  -DisplayName "Portainer HTTPS" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 9443 `
  -Action Allow

New-NetFirewallRule `
  -DisplayName "Portainer Agent" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 8000 `
  -Action Allow
```

## Step 6: Access Portainer

```powershell
# Get server IP
(Get-NetIPAddress -AddressFamily IPv4 -InterfaceAlias "Ethernet*").IPAddress

# Access URL
Write-Host "Portainer URL: https://$(hostname):9443"
```

## Step 7: Set Up as Windows Service (Auto-Start)

Docker's `--restart=always` handles container restart, but ensure Docker itself starts with Windows:

```powershell
# Docker Desktop starts with Windows automatically if configured
# For Docker Engine service:
Get-Service docker
Set-Service -Name docker -StartupType Automatic
```

## Running Windows Containers in Portainer

Portainer also supports native Windows containers:

```powershell
# Switch to Windows containers mode
# Right-click Docker Desktop -> "Switch to Windows containers"

# Windows container example
docker run -d --name iis mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022

# Portainer will show Windows containers alongside Linux containers
```

## Troubleshooting on Windows Server

### Docker Socket Path Issue

```powershell
# If Portainer cannot connect, try named pipe path
docker run -d `
  -v //./pipe/docker_engine://./pipe/docker_engine `
  portainer/portainer-ce:latest
```

### Hyper-V Not Available

Some virtualized environments don't support nested virtualization (Hyper-V inside a VM). In that case, you cannot run Linux containers on Windows Server without Hyper-V.

## Conclusion

Running Portainer on Windows Server requires Docker with Linux container support via Hyper-V. While more complex than Linux-based deployments, Windows Server + Portainer provides a familiar management interface for Windows-centric organizations. For new deployments, consider whether a Linux-based Docker host might better suit your needs, as Linux provides a more native container experience.
