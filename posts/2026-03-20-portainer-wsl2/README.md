# How to Install Portainer on WSL2 with Ubuntu - Part 2

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, WSL2, Window, Ubuntu, Docker

Description: Learn how to install Portainer inside WSL2 with Ubuntu to manage Docker containers on your Windows development machine without installing Docker Desktop.

## Why Portainer on WSL2?

WSL2 (Windows Subsystem for Linux 2) runs a real Linux kernel, making it possible to run Docker Engine natively without Docker Desktop. Portainer then provides a web-based management interface for your WSL2 Docker environment.

## Prerequisites

- Windows 10 (21H2+) or Windows 11
- WSL2 enabled
- Ubuntu 22.04 installed from Microsoft Store

## Step 1: Enable WSL2

```powershell
# In PowerShell as Administrator

wsl --install -d Ubuntu-22.04

# Ensure WSL2 is the default
wsl --set-default-version 2

# Verify
wsl --list --verbose
```

## Step 2: Install Docker Engine in WSL2 Ubuntu

```bash
# Inside WSL2 Ubuntu terminal

# Remove any old Docker installations
sudo apt remove docker docker-engine docker.io containerd runc

# Install Docker Engine
curl -fsSL https://get.docker.com | sh

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Start Docker (WSL2 doesn't use systemd by default on older versions)
sudo service docker start

# Enable systemd in WSL2 (Ubuntu 22.04+)
# Add to /etc/wsl.conf:
sudo tee -a /etc/wsl.conf << 'EOF'
[boot]
systemd=true
EOF

# Restart WSL2
# In PowerShell: wsl --shutdown
# Then reopen Ubuntu
```

## Step 3: Auto-Start Docker

```bash
# With systemd enabled:
sudo systemctl enable docker
sudo systemctl start docker

# Without systemd (older setup), add to ~/.bashrc:
echo 'if ! service docker status > /dev/null 2>&1; then sudo service docker start; fi' >> ~/.bashrc
```

## Step 4: Install Portainer

```bash
# Create Portainer volume
docker volume create portainer_data

# Run Portainer
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

echo "Portainer available at https://localhost:9443"
```

## Step 5: Access Portainer from Windows

WSL2 network is accessible from Windows via `localhost`:

Open `https://localhost:9443` in your Windows browser.

## Step 6: Configure WSL2 Auto-Start

For Portainer to be available when Windows starts:

1. Create a startup script in `%USERPROFILE%\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\`

```powershell
# wsl-docker-start.ps1 (add to Windows startup)
wsl -d Ubuntu -- sudo service docker start
wsl -d Ubuntu -- docker start portainer
```

Or use Task Scheduler:

```powershell
$action = New-ScheduledTaskAction -Execute "wsl" -Argument "-d Ubuntu -- bash -c 'sudo service docker start && docker start portainer'"
$trigger = New-ScheduledTaskTrigger -AtLogOn
Register-ScheduledTask -Action $action -Trigger $trigger -TaskName "WSL2-Portainer" -RunLevel Highest
```

## WSL2 Networking Notes

| Access | Address |
|--------|---------|
| From Windows browser | `http://localhost:9000` or `https://localhost:9443` |
| From WSL2 terminal | `http://localhost:9000` |
| From other Windows apps | `http://localhost:9000` |
| From external network | Requires port forwarding setup |

## Port Forwarding to External Network (Optional)

```powershell
# In PowerShell as Administrator - forward WSL2 port to Windows host
$wslIP = (wsl hostname -I).trim().split(" ")[0]
netsh interface portproxy add v4tov4 listenport=9443 listenaddress=0.0.0.0 connectport=9443 connectaddress=$wslIP
```

## Conclusion

Portainer in WSL2 gives Windows developers a native Linux Docker environment with a proper web UI, without requiring Docker Desktop's license. The combination of WSL2's Linux kernel, Docker Engine, and Portainer creates a developer-friendly container environment that feels identical to a Linux server.
