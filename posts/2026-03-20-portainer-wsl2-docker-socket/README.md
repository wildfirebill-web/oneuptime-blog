# How to Troubleshoot WSL2 Docker Socket Issues with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, WSL2, Docker, Troubleshooting, Windows, Docker Socket

Description: Diagnose and fix common Docker socket issues that prevent Portainer from working correctly in WSL2 environments on Windows.

## Introduction

WSL2 introduces some unique networking and socket challenges that can prevent Portainer from connecting to Docker properly. This guide covers the most common issues and their solutions, including socket permission errors, daemon connectivity problems, and WSL2-specific networking quirks.

## Common Error Messages

- `Cannot connect to the Docker daemon. Is the docker daemon running?`
- `Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/json": dial unix /var/run/docker.sock: connect: permission denied`
- `Error response from daemon: client version 1.41 is too old`
- Portainer shows "Connection error" or blank container list

## Issue 1: Docker Socket Permission Denied

**Symptom**: Portainer container starts but shows connection errors.

**Diagnosis**:

```bash
# Check socket permissions

ls -la /var/run/docker.sock
# Should show: srw-rw---- 1 root docker ...

# Check if your user is in docker group
groups
# Should include: docker
```

**Fix**:

```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Apply immediately without logout
newgrp docker

# Verify
docker ps  # Should work without sudo

# If socket is owned by wrong group
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
```

## Issue 2: Docker Daemon Not Running in WSL2

**Symptom**: Docker commands fail with "cannot connect to daemon".

**Diagnosis**:

```bash
# Check if Docker daemon is running
service docker status
# or
systemctl status docker  # If systemd is enabled
```

**Fix Without Systemd**:

```bash
# Start Docker daemon manually
sudo service docker start

# Add to .bashrc for automatic start
cat >> ~/.bashrc << 'EOF'
if ! service docker status > /dev/null 2>&1; then
    sudo service docker start > /dev/null 2>&1
fi
EOF
```

**Fix With Systemd (Ubuntu 22.04+ in WSL2)**:

```bash
# Enable systemd in WSL2
sudo tee /etc/wsl.conf > /dev/null << 'EOF'
[boot]
systemd=true
EOF

# Restart WSL2 from PowerShell
# wsl --shutdown
# Reopen Ubuntu

# Now enable Docker service
sudo systemctl enable --now docker
```

## Issue 3: Portainer Cannot Connect to Docker Socket

**Symptom**: Portainer starts but cannot list containers.

**Diagnosis**:

```bash
# Test socket connectivity from within Portainer container
docker exec portainer curl --unix-socket /var/run/docker.sock \
    http://localhost/v1.41/containers/json

# Check Portainer logs
docker logs portainer --tail 50
```

**Fix**:

Ensure the socket is properly mounted when running Portainer:

```bash
# Remove and recreate Portainer with correct socket mount
docker stop portainer && docker rm portainer

docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Issue 4: WSL2 Clock Skew

WSL2 can drift from the Windows clock, causing TLS certificate validation failures:

**Symptom**: SSL errors or 401/403 errors in Portainer.

**Fix**:

```bash
# Sync clock
sudo hwclock -s
# or
sudo ntpdate pool.ntp.org
# or
sudo timedatectl set-ntp true
```

Add to a cron job or WSL startup script:

```bash
# In /etc/wsl.conf
[boot]
command = "hwclock -s"
```

## Issue 5: WSL2 Network Interface Issues

**Symptom**: Portainer is running but not accessible from Windows browser.

**Diagnosis**:

```bash
# Check WSL2 IP address
ip addr show eth0 | grep "inet "
# WSL2 IP is usually 172.x.x.x

# Check if port is listening
ss -tlnp | grep 9000
```

**Fix for Port Forwarding**:

In WSL2, localhost forwarding should work automatically. If not:

```powershell
# In PowerShell on Windows
# Set up port proxy if WSL2 localhost forwarding isn't working
netsh interface portproxy add v4tov4 listenport=9000 listenaddress=0.0.0.0 connectport=9000 connectaddress=$(wsl hostname -I)
```

## Issue 6: Docker Desktop vs Docker Engine Conflicts

If both Docker Desktop and Docker Engine are installed:

```bash
# Check which Docker CLI is being used
which docker
# Should be /usr/bin/docker (engine) or /mnt/c/.../docker (desktop)

# Check context
docker context ls

# Use WSL2 context if using Docker Desktop
docker context use desktop-linux
```

## Issue 7: Portainer Version Mismatch with Docker API

```bash
# Check Docker API version
docker version | grep -A3 "Server:"

# Check minimum Portainer-supported Docker version
# Portainer CE requires Docker Engine 19.03+

# If using old Docker version, update
curl -fsSL https://get.docker.com | sh
```

## Diagnostic Script

Run this script to diagnose common issues:

```bash
#!/bin/bash
echo "=== WSL2 Docker/Portainer Diagnostic ==="

echo -e "\n--- Docker Daemon Status ---"
service docker status 2>&1 | head -5

echo -e "\n--- Docker Socket ---"
ls -la /var/run/docker.sock 2>&1

echo -e "\n--- Current User Groups ---"
groups

echo -e "\n--- Docker Version ---"
docker version 2>&1 | head -10

echo -e "\n--- Portainer Container Status ---"
docker ps -a --filter name=portainer --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

echo -e "\n--- Portainer Logs (last 20 lines) ---"
docker logs portainer 2>&1 | tail -20

echo -e "\n--- Port Listening Check ---"
ss -tlnp | grep -E "9000|9443"
```

## Conclusion

WSL2 Docker socket issues with Portainer are almost always related to daemon startup, socket permissions, or port forwarding. The systematic diagnostic approach - checking daemon status, socket permissions, and network connectivity in that order - resolves the vast majority of issues. Enabling systemd in WSL2 Ubuntu 22.04+ provides the most reliable Docker startup experience.
