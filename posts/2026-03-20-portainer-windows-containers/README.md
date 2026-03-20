# How to Manage Windows Containers with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Windows Containers, Docker, Windows Server, Self-Hosted, Enterprise

Description: Use Portainer to deploy and manage native Windows containers on Windows Server 2022 for .NET applications and Windows-specific workloads.

## Introduction

Windows containers allow you to run Windows-based applications (.NET Framework, IIS, Windows services) in Docker containers. Portainer supports Windows containers and provides a familiar web interface for managing them. This guide covers deploying and managing Windows containers through Portainer.

## Prerequisites

- Windows Server 2022 with Docker in Windows containers mode
- Portainer deployed (see Windows Server 2022 Portainer guide)
- Sufficient disk space (Windows container images are 3-8GB)

## Understanding Windows Container Base Images

Microsoft provides several base images:

| Image | Size | Use Case |
|-------|------|---------|
| `servercore:ltsc2022` | ~4GB | Most Windows apps, .NET Framework |
| `nanoserver:ltsc2022` | ~400MB | .NET 6+, small footprint |
| `windows:ltsc2022` | ~10GB | Apps needing full Windows APIs |
| `aspnet:latest` | ~400MB | ASP.NET Core apps |
| `dotnet/runtime:latest` | ~200MB | .NET runtime only |

## Step 1: Switch Docker to Windows Containers Mode

```powershell
# Switch Docker to Windows containers mode
& "C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon

# Verify mode
docker info | findstr "OS/Arch"
# Should show: OS/Arch: windows/amd64
```

Restart Portainer after switching:

```powershell
docker restart portainer
```

## Step 2: Deploy a .NET Framework Application

In Portainer, create a new stack:

```yaml
# Note: Windows containers use different syntax
version: "3.8"

services:
  # IIS-hosted ASP.NET application
  aspnet-app:
    image: mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022
    ports:
      - "80:80"
    volumes:
      - C:/inetpub/wwwroot:C:/inetpub/wwwroot
    restart: unless-stopped

  # SQL Server for Windows
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: "Y"
      SA_PASSWORD: "YourStrong@Passw0rd"
      MSSQL_PID: Developer
    ports:
      - "1433:1433"
    volumes:
      - sqldata:C:/var/opt/mssql
    restart: unless-stopped

volumes:
  sqldata:
```

## Step 3: Windows-Specific Volume Paths

Windows containers use Windows-style paths:

```yaml
volumes:
  # Windows volume paths use backslash notation in YAML
  - C:/app/data:C:/app/data
  - C:/logs:C:/app/logs

  # Named volumes also work
  - appdata:C:/app/data
```

## Step 4: Windows Container Environment Variables

```yaml
services:
  myapp:
    image: mcr.microsoft.com/windows/servercore:ltsc2022
    environment:
      # Windows environment variable style works fine
      - APP_ENV=production
      - DB_SERVER=mssql
      - COMPUTERNAME=container-host  # Can override Windows env vars
```

## Step 5: Deploy ASP.NET Core on Nano Server

```yaml
version: "3.8"

services:
  # Modern .NET application on Nano Server (lightweight)
  dotnet-app:
    image: mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022
    ports:
      - "5000:80"
    environment:
      - ASPNETCORE_URLS=http://+:80
      - ASPNETCORE_ENVIRONMENT=Production
    volumes:
      - C:/app:C:/app
    command: ["C:/app/MyWebApp.exe"]
    restart: unless-stopped
```

## Step 6: Windows Container Networking

Windows containers use different network drivers:

```yaml
networks:
  # nat: default for isolated containers
  appnet:
    driver: nat

  # transparent: direct access to physical network
  physicalnet:
    driver: transparent
```

## Step 7: Running PowerShell in Windows Containers

Access container console via Portainer:

1. Click on a running container
2. Click **Console**
3. Enter command: `powershell`
4. Click **Connect**

Or via command line:

```powershell
# Run PowerShell in a Windows container
docker exec -it <container-name> powershell
```

## Step 8: Building Custom Windows Container Images

```dockerfile
# Dockerfile for a Windows Server Core app
FROM mcr.microsoft.com/windows/servercore:ltsc2022

# Install Chocolatey for package management
RUN powershell -Command Set-ExecutionPolicy Bypass -Scope Process -Force; \
    [System.Net.ServicePointManager]::SecurityProtocol = 3072; \
    iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

# Install application via Chocolatey
RUN choco install -y notepadplusplus

# Copy application files
COPY myapp/ C:/app/

WORKDIR C:/app

CMD ["C:/app/myapp.exe"]
```

## Windows Container Limitations

- **No Linux containers**: When Docker is in Windows containers mode, you cannot run Linux containers
- **Image size**: Windows base images are 400MB-10GB (vs Linux's typical 10-200MB)
- **Pull time**: First pull of Windows images is slow due to their size
- **Process isolation vs Hyper-V isolation**: Default is process isolation (less secure), Hyper-V isolation provides better isolation but uses more resources

## Conclusion

Portainer's Windows container support makes it straightforward to manage legacy .NET Framework applications, SQL Server, and other Windows workloads in containers. While Windows containers have higher resource requirements than Linux containers, they enable containerization of applications that have Windows-specific dependencies, bridging the gap between traditional Windows deployments and modern container infrastructure.
