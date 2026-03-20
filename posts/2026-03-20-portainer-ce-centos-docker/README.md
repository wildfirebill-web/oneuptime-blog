# Installing Portainer CE on CentOS with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, CentOS, Docker, Self-Hosted, Container Management

Description: A step-by-step guide to installing Portainer CE on CentOS Linux with Docker to manage containers through a web-based UI.

## Prerequisites

- CentOS Linux (CentOS Stream 8 or 9)
- Root or sudo access
- Internet connectivity

## Step 1: Update the System

```bash
sudo dnf update -y
```

## Step 2: Install Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable --now docker
```

Verify:

```bash
docker --version
docker info
```

## Step 3: Create a Portainer Volume

```bash
docker volume create portainer_data
```

## Step 4: Deploy Portainer CE

```bash
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Open Firewall Ports

```bash
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --reload
```

## Step 6: Access Portainer

Navigate to `https://<server-ip>:9443` in your browser. Accept the self-signed certificate and create your admin account.

## Troubleshooting

**Permission denied on socket:**
```bash
sudo chmod 666 /var/run/docker.sock
```

**Check logs:**
```bash
docker logs portainer
```

**SELinux issues:**
```bash
sudo setsebool -P container_manage_cgroup 1
```

## Updating Portainer

```bash
docker stop portainer && docker rm portainer
docker pull portainer/portainer-ce:latest
# Re-run the deploy command
```

## Conclusion

Portainer CE on CentOS provides a powerful web-based interface for Docker container management. The installation process takes only a few minutes, and Portainer's persistent data volume ensures your configuration survives container restarts and updates.
