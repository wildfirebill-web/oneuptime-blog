# How to Run Portainer Edge Agent on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, Windows, Docker Desktop, Edge Computing

Description: Deploy the Portainer Edge Agent on Windows hosts using Docker Desktop to enable remote management of Windows-based Docker environments from a central Portainer server.

## Introduction

Windows environments running Docker Desktop or Docker Engine can be managed remotely via the Portainer Edge Agent. This is particularly useful for managing Windows workstations, Windows Server hosts, or developer machines from a central Portainer instance without requiring inbound network access. The Edge Agent initiates outbound connections, making it firewall-friendly in corporate environments.

## Prerequisites

- Windows 10/11 or Windows Server 2019/2022
- Docker Desktop for Windows or Docker Engine installed
- Portainer Business Edition (Edge Agent requires BE for full functionality)
- PowerShell 5.1+ or PowerShell Core
- Network access from the Windows host to the Portainer server on port 8000 (tunnel) and 9443 (HTTPS)

## Step 1: Generate an Edge Environment in Portainer

Navigate to **Environments** → **Add environment** → **Docker Standalone** → **Edge Agent**.

Fill in:
- **Name**: `windows-workstation-01`
- **Portainer server URL**: `https://portainer.example.com`

Click **Create** and copy the Edge ID and Edge Key shown on the screen.

## Step 2: Deploy the Edge Agent with Docker Desktop

Open PowerShell as Administrator on the Windows host.

```powershell
# Set your Edge ID and Edge Key

$EDGE_ID = "your-edge-id-here"
$EDGE_KEY = "your-edge-key-here"

# Pull the Portainer Edge Agent image
docker pull portainer/agent:latest

# Run the Edge Agent container
docker run -d `
  -v /var/run/docker.sock:/var/run/docker.sock `
  -v /var/lib/docker/volumes:/var/lib/docker/volumes `
  -v C:/ProgramData/docker:/host/docker:ro `
  -e EDGE=1 `
  -e EDGE_ID=$EDGE_ID `
  -e EDGE_KEY=$EDGE_KEY `
  -e EDGE_INSECURE_POLL=0 `
  --name portainer_edge_agent `
  --restart always `
  portainer/agent:latest
```

On Windows with Docker Desktop using Windows Containers mode, mount the named pipe:

```powershell
docker run -d `
  -v //./pipe/docker_engine://./pipe/docker_engine `
  -e EDGE=1 `
  -e EDGE_ID=$EDGE_ID `
  -e EDGE_KEY=$EDGE_KEY `
  --name portainer_edge_agent `
  --restart always `
  portainer/agent:latest
```

## Step 3: Deploy via Docker Compose on Windows

Create `C:\portainer\docker-compose.yml`:

```yaml
version: "3.8"

services:
  portainer_edge_agent:
    image: portainer/agent:latest
    container_name: portainer_edge_agent
    restart: always
    environment:
      EDGE: "1"
      EDGE_ID: "${EDGE_ID}"
      EDGE_KEY: "${EDGE_KEY}"
      EDGE_INSECURE_POLL: "0"
      EDGE_PING_INTERVAL: "5"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
```

Create `C:\portainer\.env`:

```text
EDGE_ID=your-edge-id-here
EDGE_KEY=your-edge-key-here
```

Deploy:

```powershell
cd C:\portainer
docker compose up -d
```

## Step 4: Run as a Windows Service with NSSM

For production deployments, run the Edge Agent as a Windows Service so it starts automatically with the system, even before Docker Desktop is opened.

```powershell
# Install NSSM (Non-Sucking Service Manager)
winget install NSSM.NSSM

# Create a startup script
$startScript = @"
docker start portainer_edge_agent 2>/dev/null || docker run -d `
  -v /var/run/docker.sock:/var/run/docker.sock `
  -v /var/lib/docker/volumes:/var/lib/docker/volumes `
  -e EDGE=1 `
  -e EDGE_ID=your-edge-id `
  -e EDGE_KEY=your-edge-key `
  --name portainer_edge_agent `
  --restart always `
  portainer/agent:latest
"@

$startScript | Out-File -FilePath "C:\portainer\start-agent.ps1"

# Register as a service
nssm install PortainerEdgeAgent powershell.exe
nssm set PortainerEdgeAgent AppParameters "-ExecutionPolicy Bypass -File C:\portainer\start-agent.ps1"
nssm set PortainerEdgeAgent Start SERVICE_AUTO_START
nssm start PortainerEdgeAgent
```

## Step 5: Configure Windows Firewall

The Edge Agent initiates outbound connections only. Ensure outbound access is allowed:

```powershell
# Allow outbound HTTPS to Portainer (port 9443)
New-NetFirewallRule `
  -DisplayName "Portainer Edge Agent HTTPS" `
  -Direction Outbound `
  -Protocol TCP `
  -RemotePort 9443 `
  -Action Allow

# Allow outbound tunnel connection (port 8000)
New-NetFirewallRule `
  -DisplayName "Portainer Edge Agent Tunnel" `
  -Direction Outbound `
  -Protocol TCP `
  -RemotePort 8000 `
  -Action Allow
```

## Step 6: Verify the Connection

Check the agent container logs in PowerShell:

```powershell
docker logs portainer_edge_agent --tail 50 -f
```

Expected output:
```text
Starting Portainer agent version X.X.X
Edge mode enabled
Polling https://portainer.example.com/api/endpoints/XX/edge/status every 5 seconds
```

In the Portainer UI, navigate to **Environments** and verify the Windows environment shows as **Heartbeat** active.

## Async Mode for Intermittently Connected Windows Hosts

For laptops or machines that may be offline periodically:

```powershell
docker run -d `
  -v /var/run/docker.sock:/var/run/docker.sock `
  -v /var/lib/docker/volumes:/var/lib/docker/volumes `
  -e EDGE=1 `
  -e EDGE_ID=$EDGE_ID `
  -e EDGE_KEY=$EDGE_KEY `
  -e EDGE_ASYNC=1 `
  -e EDGE_PING_INTERVAL=60 `
  -e EDGE_CMD_INTERVAL=60 `
  -e EDGE_SNAPSHOT_INTERVAL=300 `
  --name portainer_edge_agent `
  --restart always `
  portainer/agent:latest
```

With async mode enabled, the agent will periodically upload environment state and download pending commands, allowing Portainer to queue deployments for when the Windows host comes back online.

## Troubleshooting

**Agent cannot connect:**
- Verify DNS resolution of the Portainer server hostname from the Windows host
- Check `Test-NetConnection portainer.example.com -Port 8000` from PowerShell
- Ensure Docker Desktop is running before the agent starts

**Docker socket not found:**
- Confirm Docker Desktop is running: `docker info`
- Check the Docker context: `docker context ls`
- Switch to the correct context if using WSL2: `docker context use default`

**Container restarts repeatedly:**
- Review full logs: `docker logs portainer_edge_agent`
- Verify the EDGE_KEY is correct and has not expired
- Check that the environment ID in Portainer matches EDGE_ID

## Conclusion

Running the Portainer Edge Agent on Windows enables centralized management of Docker workloads across Windows hosts without requiring VPN or inbound firewall rules. The outbound-only architecture is well-suited for corporate Windows environments where IT policy restricts inbound connections. For maximum reliability on Windows, combine Docker Desktop with a startup script registered as a Windows Service so the agent survives reboots and system events automatically.
