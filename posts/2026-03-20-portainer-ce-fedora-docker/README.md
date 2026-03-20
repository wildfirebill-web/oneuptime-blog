# Installing Portainer CE on Fedora with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Fedora, Docker, Self-Hosted, Container Management

Description: A step-by-step guide to installing Portainer CE on Fedora Linux with Docker to manage containers through a web-based UI.

## Prerequisites

- Fedora 38 or later
- Root or sudo access
- Internet connectivity

## Step 1: Update the System

```bash
sudo dnf update -y
```

## Step 2: Install Docker

Fedora ships with Podman by default. To install Docker Engine:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable --now docker
```

> **Note:** If Podman conflicts with Docker, you may need to remove Podman first: `sudo dnf remove podman`

Verify:

```bash
docker --version
docker run hello-world
```

## Step 3: Create the Portainer Volume

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

Navigate to `https://<server-ip>:9443` and set up your admin account.

## Troubleshooting

**SELinux blocking Docker socket access:**
```bash
sudo setsebool -P container_manage_cgroup 1
sudo chcon -t container_file_t /var/run/docker.sock
```

**Docker socket permission:**
```bash
sudo chmod 666 /var/run/docker.sock
```

**Check logs:**
```bash
docker logs portainer
```

## Updating Portainer

```bash
docker stop portainer && docker rm portainer
docker pull portainer/portainer-ce:latest
# Re-run the deploy command
```

## Conclusion

Portainer CE on Fedora gives you a modern web UI for Docker container management. Note the Podman compatibility consideration on Fedora — once Docker is properly configured, Portainer runs reliably with the `--restart=always` policy.
