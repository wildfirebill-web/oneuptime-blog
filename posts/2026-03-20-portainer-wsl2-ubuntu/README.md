# How to Install Portainer on WSL2 with Ubuntu - Part 3

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, WSL2, Ubuntu, Docker, Window, Self-Hosted, Development

Description: Run Portainer inside WSL2 on Windows 10/11 using Docker Engine directly in Ubuntu, without Docker Desktop, for a lightweight development container management setup.

## Introduction

WSL2 (Windows Subsystem for Linux 2) lets you run a full Linux environment inside Windows. Installing Docker Engine directly in WSL2 Ubuntu and running Portainer gives you a lightweight container management setup without requiring Docker Desktop. This is particularly useful for development environments and home labs on Windows machines.

## Prerequisites

- Windows 10 version 2004+ or Windows 11
- WSL2 enabled with Ubuntu 22.04 or 24.04
- 8GB RAM recommended

## Step 1: Install and Configure WSL2

```powershell
# In PowerShell (Admin), install WSL2 with Ubuntu

wsl --install -d Ubuntu-22.04

# Set WSL2 as default version
wsl --set-default-version 2

# Verify Ubuntu is using WSL2
wsl -l -v
# Should show:  Ubuntu-22.04  Running  2
```

## Step 2: Configure WSL2 Resources

Create or edit `%USERPROFILE%\.wslconfig` in Windows:

```ini
[wsl2]
# Allocate up to 8GB RAM (adjust based on your system)
memory=8GB
# Use up to 4 processor cores
processors=4
# Swap file size
swap=2GB
# Enable localhost forwarding
localhostforwarding=true

[experimental]
# Automatically free memory when WSL2 is idle
autoMemoryReclaim=dropcache
networkingMode=mirrored
```

Apply changes:

```powershell
wsl --shutdown
# Then reopen Ubuntu
```

## Step 3: Update Ubuntu and Install Dependencies

Inside WSL2 Ubuntu:

```bash
# Update Ubuntu
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    iptables
```

## Step 4: Configure iptables for Docker

WSL2 Ubuntu uses iptables-nft by default, but Docker works better with iptables-legacy:

```bash
# Switch to iptables-legacy
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy

# Verify
sudo iptables --version
# Should show: iptables v1.8.x (legacy)
```

## Step 5: Install Docker Engine

```bash
# Install Docker Engine (not Docker Desktop)
curl -fsSL https://get.docker.com | sh

sudo usermod -aG docker $USER
newgrp docker
```

## Step 6: Start Docker in WSL2

Docker daemon doesn't start automatically in WSL2. Add startup to `.bashrc` or `.profile`:

```bash
# Add to ~/.bashrc
cat >> ~/.bashrc << 'EOF'

# Start Docker daemon if not running
if ! service docker status > /dev/null 2>&1; then
    sudo service docker start
fi
EOF

source ~/.bashrc
```

Or use systemd (WSL2 with Ubuntu 22.04+):

```bash
# Enable systemd in WSL2 (Ubuntu 22.04+)
sudo tee /etc/wsl.conf > /dev/null << 'EOF'
[boot]
systemd=true
EOF

# Restart WSL (from PowerShell)
# wsl --shutdown
# Then reopen Ubuntu - Docker will start automatically
```

## Step 7: Deploy Portainer

```bash
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 8: Access Portainer from Windows

With WSL2's localhost forwarding, access directly from Windows browser:

```text
http://localhost:9000
```

No additional configuration needed - WSL2 automatically forwards ports to Windows.

## Step 9: Windows Terminal Integration

Add a WSL2 profile to Windows Terminal for easy access:

```json
{
  "name": "Ubuntu-Docker",
  "commandline": "wsl -d Ubuntu-22.04 -- bash -c 'cd ~ && bash'",
  "icon": "https://assets.ubuntu.com/v1/49a1a858-favicon-32x32.png"
}
```

## Persistent Docker Startup with Task Scheduler

To start WSL2 and Docker automatically at Windows logon:

```powershell
# Create a VBScript to start WSL silently
Set-Content -Path "$env:APPDATA\StartWSLDocker.vbs" -Value @'
Set objShell = CreateObject("WScript.Shell")
objShell.Run "wsl -d Ubuntu-22.04 -- bash -c 'sudo service docker start'", 0, False
'@

# Register as Task Scheduler task at logon
$action = New-ScheduledTaskAction -Execute "wscript.exe" -Argument "$env:APPDATA\StartWSLDocker.vbs"
$trigger = New-ScheduledTaskTrigger -AtLogon
Register-ScheduledTask -TaskName "Start WSL Docker" -Action $action -Trigger $trigger
```

## Conclusion

WSL2 with Docker Engine and Portainer provides a full Linux container development environment inside Windows without Docker Desktop's licensing requirements. The WSL2 localhost forwarding makes Portainer accessible directly from Windows browsers. With systemd enabled, Docker starts automatically, making this a seamless integrated development environment.
