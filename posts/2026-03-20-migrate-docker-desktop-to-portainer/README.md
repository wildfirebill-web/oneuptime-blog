# How to Migrate from Docker Desktop to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Desktop, Migration, Linux, Containers

Description: Step-by-step guide to replacing Docker Desktop with Portainer for container management on development and production systems.

## Introduction

Docker Desktop requires a commercial license for large organizations and runs as a VM on macOS and Windows. Portainer, combined with the Docker engine directly, offers a free, lightweight alternative that works natively on Linux servers. This guide covers migrating workloads and workflows from Docker Desktop to Portainer.

## Why Migrate?

- Docker Desktop has commercial licensing requirements for companies over 250 employees or $10M revenue
- Docker Desktop on macOS/Windows runs containers in a VM, adding overhead
- Portainer works with the native Docker engine on Linux, eliminating VM overhead
- Portainer offers superior team access control and audit logging

## Step 1: Export Your Docker Desktop Data

Before migrating, export important configurations:

```bash
# List all images to migrate
docker images --format "{{.Repository}}:{{.Tag}}" > images-to-migrate.txt

# List all volumes
docker volume ls --format "{{.Name}}" > volumes-to-migrate.txt

# Export each image
while read image; do
  filename=$(echo $image | tr '/:' '--')
  docker save $image -o "exports/$filename.tar"
done < images-to-migrate.txt

# Export volumes
mkdir -p volume-exports
while read vol; do
  docker run --rm \
    -v $vol:/data \
    -v $(pwd)/volume-exports:/backup \
    alpine tar czf /backup/$vol.tar.gz -C /data .
  echo "Exported volume: $vol"
done < volumes-to-migrate.txt

# Export docker-compose files
find . -name "docker-compose*.yml" -exec cp {} compose-exports/ \;
```

## Step 2: Set Up Linux Server with Docker Engine

```bash
# On your Linux server (Ubuntu/Debian)
curl -fsSL https://get.docker.com | sh

# Add your user to the docker group
sudo usermod -aG docker $USER

# Enable Docker to start on boot
sudo systemctl enable --now docker

# Verify
docker --version
docker info
```

## Step 3: Install Portainer

```bash
# Create Portainer data volume
docker volume create portainer_data

# Deploy Portainer
docker run -d \
  --name portainer \
  --restart=always \
  -p 8000:8000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Access at https://your-server-ip:9443
```

## Step 4: Import Images and Volumes

```bash
# Copy exported files to the Linux server
rsync -av exports/ user@linux-server:/tmp/docker-imports/
rsync -av volume-exports/ user@linux-server:/tmp/volume-imports/

# On the Linux server: Import images
for tar in /tmp/docker-imports/*.tar; do
  docker load -i $tar
  echo "Loaded: $tar"
done

# Import volumes
for archive in /tmp/volume-imports/*.tar.gz; do
  vol_name=$(basename $archive .tar.gz)
  docker volume create $vol_name
  docker run --rm \
    -v $vol_name:/data \
    -v /tmp/volume-imports:/backup \
    alpine tar xzf /backup/$(basename $archive) -C /data
  echo "Imported volume: $vol_name"
done
```

## Step 5: Migrate Docker Compose Stacks

Import your compose files into Portainer:

```bash
# Copy compose files to the server
scp -r compose-exports/ user@linux-server:/opt/stacks/

# In Portainer UI:
# Stacks > Add Stack > Web editor
# Paste your docker-compose.yml content
# Or: Upload button to upload the file directly
```

## Step 6: Configure Remote Access

Replace Docker Desktop's local GUI with Portainer accessible remotely:

```bash
# Option 1: Access via browser
# https://your-server-ip:9443

# Option 2: SSH tunnel for secure access
ssh -L 9443:localhost:9443 user@your-server

# Option 3: Set up Cloudflare Tunnel or Tailscale for secure access
# Install Tailscale on the server
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# Access via: https://tailscale-ip:9443
```

## Docker Context Migration (for CLI Users)

```bash
# Add the remote Docker host as a context
docker context create linux-server \
  --docker "host=ssh://user@your-server"

# Switch to the new context
docker context use linux-server

# Verify
docker context ls
docker ps  # Now manages remote containers
```

## Conclusion

Migrating from Docker Desktop to Portainer eliminates licensing concerns and VM overhead while providing superior team collaboration features. The migration process preserves all your images, volumes, and compose configurations. Portainer's web UI replaces Docker Desktop's GUI with a more feature-rich, server-ready interface accessible from any browser.
