# How to Switch Between Linux and Windows Containers in Portainer - Switch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Windows, Linux Containers, Docker, Container Mode

Description: Learn how to switch between Linux and Windows container modes on Windows hosts and how this affects your Portainer environment and running containers.

## Understanding Container Modes

On Windows, Docker can run in two modes:

- **Linux container mode**: Uses a lightweight Linux VM (Hyper-V or WSL2) to run Linux images
- **Windows container mode**: Runs native Windows container images on the Windows kernel

You can only run one mode at a time, and switching requires restarting Docker.

## How to Switch Container Modes

### Docker Desktop (Windows 10/11)

**Method 1**: Right-click the Docker Desktop tray icon

- If "Switch to Windows containers..." appears → you're in Linux mode
- If "Switch to Linux containers..." appears → you're in Windows mode

**Method 2**: Via command line

```powershell
# Switch to Windows containers

& "$Env:ProgramFiles\Docker\Docker\DockerCli.exe" -SwitchWindowsEngine

# Switch to Linux containers
& "$Env:ProgramFiles\Docker\Docker\DockerCli.exe" -SwitchLinuxEngine
```

### Docker on Windows Server

```powershell
# Check current mode
docker version | grep "OS/Arch"
# linux/amd64 = Linux mode
# windows/amd64 = Windows mode

# Switch via configuration (requires restart)
# Edit C:\ProgramData\Docker\config\daemon.json
# Add: "experimental": true (if needed for your version)
# Then configure isolation level
```

## What Happens to Portainer When You Switch?

When you switch container modes, **running containers are not accessible** in the new mode. Portainer itself must be restarted to reconnect to the new Docker engine context.

```powershell
# 1. Stop Portainer
docker stop portainer

# 2. Switch container mode
& "$Env:ProgramFiles\Docker\Docker\DockerCli.exe" -SwitchWindowsEngine

# 3. Restart Portainer (using Windows container mode syntax)
docker start portainer
# OR create a new Portainer container appropriate for Windows containers
docker run -d `
  -p 9000:9000 `
  -p 8000:8000 `
  --name portainer-win `
  --restart always `
  -v \\.\pipe\docker_engine:\\.\pipe\docker_engine `
  -v portainer_data:C:\data `
  portainer/portainer-ce:latest
```

## Recommended Pattern: Two Portainer Instances

Run separate Portainer instances for each mode:

```powershell
# Portainer for Linux containers (port 9000)
docker run -d `
  -p 9000:9000 `
  --name portainer-linux `
  -v \\.\pipe\docker_engine:\\.\pipe\docker_engine `
  -v portainer_linux_data:C:\data `
  portainer/portainer-ce:latest

# Portainer for Windows containers (port 9001)
docker run -d `
  -p 9001:9000 `
  --name portainer-windows `
  -v \\.\pipe\docker_engine:\\.\pipe\docker_engine `
  -v portainer_windows_data:C:\data `
  portainer/portainer-ce:latest
```

Access:
- Linux containers: `http://localhost:9000`
- Windows containers: `http://localhost:9001`

## Verifying Current Mode

```powershell
# Check Docker mode
docker version --format '{{.Server.Os}}'
# Outputs: linux or windows

# Check available images by architecture
docker images --format "{{.Repository}}:{{.Tag}} {{.ID}}"

# Check if a specific image is available for current mode
docker manifest inspect nginx:alpine | Select-String "os"
```

## Container Compatibility Reference

| Image Type | Linux Mode | Windows Mode |
|-----------|-----------|-------------|
| `nginx:alpine` | ✓ Works | ✗ Fails |
| `mcr.microsoft.com/windows/servercore/iis` | ✗ Fails | ✓ Works |
| `mcr.microsoft.com/dotnet/aspnet:8.0` | ✓ Works | ✓ Works (different tag) |

## Conclusion

Switching between Linux and Windows container modes is a Docker-level operation that affects Portainer's view of the environment. For teams that need both container types, either maintain separate Portainer instances or use a Portainer Business Edition environment that spans both contexts. The most common setup for development is Linux containers via WSL2 (no mode switching needed), with Windows container mode used only for Windows-specific workloads.
