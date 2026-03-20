# How to Fix 'Portainer UI Not Accessible After Installation'

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Self-Hosted

Description: Diagnose and fix the most common reasons why the Portainer web UI is not accessible immediately after installation.

## Introduction

You've installed Portainer, but navigating to port 9000 or 9443 gives you a connection timeout or refused error. This is one of the most common issues new users face. This guide covers every likely cause and its fix.

## Common Causes

1. Container not running
2. Wrong port binding
3. Firewall blocking the port
4. Initialization timeout (5-minute admin setup window)
5. HTTPS-only mode with no certificate
6. Listening on wrong interface

## Step 1: Verify the Container Is Running

```bash
# Check Portainer container status

docker ps | grep portainer

# If not listed, check all containers including stopped ones
docker ps -a | grep portainer

# If stopped, check the exit code and logs
docker logs portainer
```

If the container exited immediately, the logs will show the error. Common log errors:

```text
# Permission denied on data volume
Error: open /data/portainer.db: permission denied

# Port already in use
Error: listen tcp 0.0.0.0:9000: bind: address already in use
```

## Step 2: Confirm Port Binding

```bash
# Check what ports Portainer is actually bound to
docker inspect portainer | grep -A 20 '"Ports"'

# Or use docker ps output
docker ps --format "table {{.Names}}\t{{.Ports}}"

# Check if the port is listening on the host
ss -tlnp | grep 9000
# or
netstat -tlnp | grep 9000
```

If nothing is listening on port 9000, the container may have crashed or the port binding was not specified correctly.

## Step 3: Check Firewall Rules

```bash
# Ubuntu/Debian (ufw)
sudo ufw status
# If active, allow port 9000
sudo ufw allow 9000/tcp
sudo ufw allow 9443/tcp

# CentOS/RHEL (firewalld)
sudo firewall-cmd --list-ports
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --reload

# Check iptables directly
sudo iptables -L INPUT -n | grep 9000
```

## Step 4: Verify the Portainer Run Command

The most common installation mistake is missing the port mapping:

```bash
# WRONG - no port mapping
docker run -d -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data portainer/portainer-ce:latest

# CORRECT - with port mapping
docker run -d -p 9000:9000 -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Re-create the Container with Correct Ports

```bash
# Stop and remove the incorrectly configured container
docker stop portainer
docker rm portainer

# Re-create with correct settings
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 6: Check for the 5-Minute Initialization Timeout

Portainer requires you to create an admin account within 5 minutes of first launch. After this window, Portainer **locks itself** and becomes inaccessible.

```bash
# Check logs for the timeout message
docker logs portainer 2>&1 | grep -i "timeout\|init\|admin"
```

If you see a timeout message, reset Portainer:

```bash
# Stop and remove the container
docker stop portainer && docker rm portainer

# Remove the data volume to start fresh
docker volume rm portainer_data

# Re-create Portainer
docker run -d -p 9000:9000 -p 9443:9443 \
  --name portainer --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

> **Warning**: Removing the data volume deletes all Portainer configuration. Only do this for a fresh installation.

## Step 7: Check If HTTPS-Only Mode Is Active

If Portainer was started with `--http-disabled`, port 9000 will not respond:

```bash
# Check startup flags
docker inspect portainer | grep -A 10 '"Cmd"'
```

If `--http-disabled` is present, use `https://your-host:9443` instead.

## Step 8: Test Connectivity

```bash
# Test from the same host
curl -v http://localhost:9000

# Test from another machine
curl -v http://your-server-ip:9000

# For HTTPS
curl -k -v https://your-server-ip:9443
```

## Step 9: Check Docker Network Interface

On some systems, Docker binds to a specific interface. Verify:

```bash
# Check Docker daemon configuration
cat /etc/docker/daemon.json

# Check if Docker is using a custom bridge
docker network inspect bridge | grep Subnet
```

## Conclusion

The most common causes of Portainer being inaccessible are: missing port mapping in the run command, a firewall blocking port 9000, or the 5-minute initialization timeout having expired. Systematically checking each of these using the commands above will identify and resolve the issue quickly.
