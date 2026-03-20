# How to Switch Between Linux and Windows Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Windows Containers, Linux Containers, Windows Server, Switching

Description: Understand and manage the process of switching between Linux and Windows container modes in Docker on Windows, and how Portainer handles both modes.

## Introduction

Docker on Windows supports two container modes: Linux containers (via WSL2 or Hyper-V) and native Windows containers. Portainer works with both, but you can only run one mode at a time on the same Docker daemon. This guide explains how to switch between modes and how to manage your Portainer setup through transitions.

## Understanding the Two Modes

**Linux Containers Mode** (default in Docker Desktop):
- Runs Linux containers via WSL2 or Hyper-V
- Most Docker Hub images work
- Better ecosystem support
- Used for web apps, databases, microservices

**Windows Containers Mode**:
- Runs native Windows containers
- Required for .NET Framework, IIS, Windows services
- Images are Windows-only
- Better Windows OS integration

## Prerequisites

- Windows 10/11 Pro or Enterprise, or Windows Server 2022
- Docker Desktop or Docker Engine with both modes available
- Portainer deployed

## Switching Container Modes

### Via Docker Desktop System Tray

1. Right-click the Docker icon in the system tray
2. Click **Switch to Windows containers** or **Switch to Linux containers**
3. Docker will restart the daemon

### Via PowerShell

```powershell
# Switch to Windows containers

& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# Verify current mode
docker info | findstr "OS/Arch"
```

### Via Docker Desktop CLI

```powershell
# Check current mode
docker info --format "{{.OSType}}"

# Switch modes
Stop-Service com.docker.service
# Edit Docker settings to change OS type
# Then restart
Start-Service com.docker.service
```

## Handling Portainer Through Mode Switches

Portainer itself is a Linux container. When you switch to Windows containers mode, Portainer stops being accessible through the Windows Docker daemon.

### Solution: Run Portainer with Separate Agents

Run a Portainer server on a separate Linux host and connect agents to both your Windows Docker instances:

```yaml
# Linux host: portainer-server
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    restart: unless-stopped

volumes:
  portainer_data:
```

On Windows (Linux containers mode):

```powershell
# Deploy Portainer Agent for Linux containers endpoint
docker run -d `
  --name portainer-agent `
  --restart=unless-stopped `
  -p 9001:9001 `
  -v //./pipe/docker_engine://./pipe/docker_engine `
  portainer/agent:latest
```

## Managing Both Modes from One Portainer

In Portainer BE (Business Edition), you can add multiple environments:

1. Navigate to **Environments > Add environment**
2. Add **Linux Agent** endpoint pointing to your Linux Docker host
3. Add **Windows Agent** endpoint pointing to Windows Docker host (Linux containers mode)

## Script for Mode Detection and Switching

```powershell
# PowerShell script to manage container mode switching
function Get-DockerMode {
    $info = docker info 2>$null | Select-String "OSType"
    if ($info -match "windows") {
        return "windows"
    } else {
        return "linux"
    }
}

function Switch-ToWindowsContainers {
    Write-Host "Switching to Windows containers..."
    
    # Stop Portainer first (it's a Linux container)
    docker stop portainer 2>$null
    
    # Switch mode
    & "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon
    
    Start-Sleep -Seconds 10
    
    Write-Host "Now in Windows containers mode."
    Write-Host "Portainer is not available in this mode."
    Write-Host "Use: docker ps to manage Windows containers directly"
}

function Switch-ToLinuxContainers {
    Write-Host "Switching to Linux containers..."
    
    & "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon
    
    Start-Sleep -Seconds 10
    
    # Restart Portainer
    docker start portainer
    
    Write-Host "Now in Linux containers mode."
    Write-Host "Portainer is available at http://localhost:9000"
}

# Main
$currentMode = Get-DockerMode
Write-Host "Current Docker mode: $currentMode"
```

## Creating Mode-Specific Docker Compose Files

Maintain separate compose files for each mode:

**`docker-compose.linux.yml`** (Linux containers):
```yaml
version: "3.8"
services:
  webapp:
    image: nginx:alpine
    ports:
      - "80:80"
```

**`docker-compose.windows.yml`** (Windows containers):
```yaml
version: "3.8"
services:
  webapp:
    image: mcr.microsoft.com/iis:latest
    ports:
      - "80:80"
```

## Best Practices

1. **Default to Linux containers** for most workloads - better image availability and smaller sizes
2. **Only switch to Windows containers** when you specifically need Windows OS APIs or .NET Framework
3. **Use a Linux-based Portainer server** with remote agents for Windows hosts
4. **Document your workflows** for teams that need to switch modes

## Conclusion

Managing container mode switching requires planning, especially when Portainer itself runs as a Linux container. The cleanest solution for organizations needing both modes is to use a dedicated Linux host for Portainer BE with remote agents connecting to Windows Docker hosts. For simpler setups, maintaining the Linux containers mode as default and switching to Windows mode only when needed works well.
