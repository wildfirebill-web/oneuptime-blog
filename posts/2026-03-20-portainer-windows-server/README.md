# How to Install Portainer on Windows Server 2022 with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Windows Server, Docker, Windows Containers

Description: Learn how to install Portainer on Windows Server 2022 to manage Docker containers running on Windows, including both Linux containers via Hyper-V and native Windows containers.

## Prerequisites

- Windows Server 2022 (or 2019)
- Administrator access
- Internet connectivity for downloading Docker

## Step 1: Install Docker on Windows Server

Docker Desktop is for desktop OS. Windows Server uses Docker Engine (Mirantis Container Runtime) or Docker's native Windows engine:

```powershell
# Install Containers and Hyper-V features
Install-WindowsFeature -Name Containers
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools

# Restart required
Restart-Computer -Force
```

After restart:

```powershell
# Install Docker using PowerShell
Install-Module DockerMsftProvider -Repository PSGallery -Force
Install-Package Docker -ProviderName DockerMsftProvider -Force

# Start Docker
Start-Service Docker

# Verify installation
docker version
```

## Step 2: Install Portainer

```powershell
# Create volume for Portainer data
docker volume create portainer_data

# Run Portainer
docker run -d `
  -p 8000:8000 `
  -p 9443:9443 `
  --name portainer `
  --restart always `
  -v \\.\pipe\docker_engine:\\.\pipe\docker_engine `
  -v portainer_data:C:\data `
  portainer/portainer-ce:latest
```

Note: Windows uses `\\.\pipe\docker_engine` instead of a Unix socket.

## Step 3: Configure Windows Firewall

```powershell
# Allow Portainer HTTPS port
New-NetFirewallRule `
  -DisplayName "Portainer HTTPS" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 9443 `
  -Action Allow

# Allow Portainer HTTP port
New-NetFirewallRule `
  -DisplayName "Portainer HTTP" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 9000 `
  -Action Allow
```

## Step 4: Access Portainer

Open a browser and navigate to:
```
https://SERVER_IP:9443
```

Complete the initial setup to create your admin account.

## Step 5: Switching Container Modes

Windows Server can run both Linux containers (via Hyper-V) and Windows containers:

```powershell
# Switch to Windows containers
& $Env:ProgramFiles\Docker\Docker\DockerCli.exe -SwitchWindowsEngine

# Switch to Linux containers (via Hyper-V)
& $Env:ProgramFiles\Docker\Docker\DockerCli.exe -SwitchLinuxEngine
```

## Running a Test Container

```powershell
# Linux container (if in Linux mode)
docker run -d -p 8080:80 --name nginx-test nginx:alpine

# Windows container (if in Windows mode)
docker run -d -p 8081:80 --name iis-test mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022
```

## Portainer with Docker Compose on Windows

```powershell
# Install Docker Compose
Invoke-WebRequest "https://github.com/docker/compose/releases/latest/download/docker-compose-windows-x86_64.exe" `
  -OutFile "$Env:ProgramFiles\Docker\docker-compose.exe"

# Use from Portainer stacks normally
```

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Named pipe error | Wrong volume mount for Docker socket | Use `\\.\pipe\docker_engine` |
| Container not starting | Image architecture mismatch | Ensure correct container mode |
| Port conflict | Windows services using port 80 | Check IIS or other services |

## Conclusion

Portainer on Windows Server 2022 provides a familiar web-based management interface for Docker on Windows. The primary difference from Linux installations is the Docker socket path (named pipe instead of Unix socket). Once running, Portainer manages both Windows and Linux containers through the same interface.
