# How to Troubleshoot Podman Socket Connection Issues in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Podman, Troubleshooting, Docker Socket, Debugging

Description: Diagnose and fix Podman socket connection failures in Portainer, covering socket permissions, service status, API compatibility issues, and rootless vs rootful configuration differences.

## Introduction

Connecting Portainer to a Podman socket can fail for several reasons: the socket service is not running, file permissions prevent access, the socket path is wrong, or the Podman API version is incompatible. This guide covers systematic troubleshooting steps to resolve all common Podman socket connection issues in Portainer.

## Common Error Messages

- `Cannot connect to the Docker daemon at unix:///var/run/docker.sock`
- `error during connect: Get "http://...": dial unix /var/run/docker.sock: no such file`
- `permission denied while trying to connect to the Docker daemon socket`
- `Error response from daemon: client version ... is too new. Maximum supported API version is ...`

## Step 1: Verify the Podman Socket Service Status

```bash
# Check rootful socket status
sudo systemctl status podman.socket

# Check rootless socket status (run as the relevant user)
systemctl --user status podman.socket

# If the socket service is not running, enable and start it
sudo systemctl enable --now podman.socket         # rootful
systemctl --user enable --now podman.socket       # rootless

# Check if the socket file exists
ls -la /run/podman/podman.sock                    # rootful
ls -la /run/user/$(id -u)/podman/podman.sock      # rootless
```

## Step 2: Test the Podman API Directly

```bash
# Test rootful socket with Docker-compatible API
curl -s --unix-socket /run/podman/podman.sock \
  http://d/v1.44/version | jq .

# Test rootless socket
curl -s --unix-socket /run/user/1000/podman/podman.sock \
  http://d/v1.44/version | jq .

# List containers to verify basic API functionality
curl -s --unix-socket /run/podman/podman.sock \
  http://d/v1.44/containers/json | jq '.[].Names'

# If you get a 404 error on /v1.44, try a lower version:
curl -s --unix-socket /run/podman/podman.sock \
  http://d/v1.41/version | jq .
```

## Step 3: Check Socket File Permissions

```bash
# Check the socket file ownership and permissions
ls -la /run/podman/podman.sock
# Example output:
# srw-rw---- 1 root root 0 Mar 20 10:00 /run/podman/podman.sock

# The socket must be readable by the user running Portainer
# Option A: Add the Portainer container user to the socket group
sudo chmod 666 /run/podman/podman.sock  # Temporary (resets on restart)

# Option B: Add user to the docker/podman group
sudo usermod -aG docker portainer-user
# Or create a podman group and update the socket
sudo groupadd podman
sudo chown root:podman /run/podman/podman.sock
sudo chmod 660 /run/podman/podman.sock

# Verify Portainer container can access the socket
docker exec portainer ls -la /var/run/docker.sock
```

## Step 4: Verify Socket Mount in Portainer Container

```bash
# Check how Portainer is mounted
docker inspect portainer | jq '.[0].Mounts'

# The output should show the Podman socket mapped to Docker socket path:
# {
#   "Source": "/run/podman/podman.sock",
#   "Destination": "/var/run/docker.sock",
#   "Mode": "rw"
# }

# If it's missing or wrong, recreate the container with correct mount:
docker stop portainer && docker rm portainer

docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /run/podman/podman.sock:/var/run/docker.sock \  # Correct path
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Resolve API Version Mismatches

```bash
# Check what API version Podman reports
curl -s --unix-socket /run/podman/podman.sock \
  http://d/v1.44/version | jq '.ApiVersion'

# Check Podman version
podman version

# Portainer requires Podman's Docker-compatible API (libpod/v4+)
# Upgrade Podman if version is too old:
sudo dnf update podman    # RHEL/Fedora
sudo apt update && sudo apt upgrade podman  # Ubuntu/Debian

# Test the minimum required endpoints:
# /v1.44/containers/json
# /v1.44/images/json
# /v1.44/info
for endpoint in containers/json images/json info; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    --unix-socket /run/podman/podman.sock "http://d/v1.44/$endpoint")
  echo "$endpoint: HTTP $STATUS"
done
```

## Step 6: Handle SELinux Blocking the Socket

```bash
# Check if SELinux is denying socket access
sudo ausearch -m AVC -ts recent | grep podman

# View SELinux denials in audit log
sudo grep "avc:  denied" /var/log/audit/audit.log | grep podman | tail -5

# If denials exist, apply a fix:

# Option A: Use a permissive label for Portainer container
docker run -d \
  --security-opt label:disable \  # Disable SELinux labels for this container
  -v /run/podman/podman.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Option B: Generate and install a custom policy
sudo ausearch -m AVC -ts recent | audit2allow -M portainer-podman
sudo semodule -i portainer-podman.pp
```

## Step 7: Debug Rootless Podman Lingering Issues

```bash
# For rootless Podman: ensure the user session persists
sudo loginctl enable-linger your-username

# Verify linger is enabled
loginctl show-user your-username | grep Linger

# Check if the systemd user session has the socket
sudo -u your-username systemctl --user status podman.socket

# The DBUS_SESSION_BUS_ADDRESS must be set for user services
# Check environment in the context where Portainer runs
systemctl --user show-environment | grep -E "XDG_RUNTIME_DIR|DBUS"
```

## Step 8: Check for Port or Firewall Blocking (Remote Podman)

```bash
# If using TCP-based Podman API
# Verify Podman is listening on the expected port
ss -tlnp | grep 2375

# Test connectivity from Portainer host
curl http://podman-host:2375/v1.44/version

# Check firewall rules
sudo firewall-cmd --list-all | grep 2375       # firewalld
sudo iptables -L -n | grep 2375                # iptables
sudo ufw status | grep 2375                    # ufw

# Open the port if blocked
sudo firewall-cmd --add-port=2375/tcp --permanent
sudo firewall-cmd --reload
```

## Step 9: View Portainer and Podman Logs for Errors

```bash
# Portainer container logs
docker logs portainer --tail 50

# Look for specific errors
docker logs portainer 2>&1 | grep -i "error\|cannot connect\|socket\|podman"

# Podman system service logs
sudo journalctl -u podman.socket -n 50
sudo journalctl -u podman.service -n 50

# Rootless logs
journalctl --user -u podman.socket -n 50
```

## Conclusion

Podman socket connection issues in Portainer typically fall into four categories: the socket service is not running, permissions prevent access, SELinux is blocking the socket, or the Podman API version is too old. Start by verifying the socket exists and the service is active, then confirm the Portainer container has the socket mounted at `/var/run/docker.sock`, and finally test the API directly with curl before restarting Portainer. For rootless Podman, always ensure linger is enabled so the socket service starts without requiring a user login session.
