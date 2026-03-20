# How to Deploy Harbor Registry via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Harbor, Container Registry, Self-Hosted

Description: Deploy Harbor, the enterprise-grade container registry, using Portainer stacks for a secure and feature-rich private image repository.

## Introduction

Harbor is an open-source container registry that extends the Docker Distribution project with enterprise features: role-based access control, image vulnerability scanning, content trust, and replication. Deploying it via Portainer simplifies the operational overhead and gives you a single pane of glass for all your container infrastructure.

## Prerequisites

- Portainer CE or BE installed and running
- Docker Compose v2 support enabled in Portainer
- Minimum 4 GB RAM on the host
- A domain name with DNS pointing to your host

## Step 1: Download Harbor Installer

Harbor recommends using its offline installer. Run these commands on the Docker host, not inside Portainer:

```bash
# Download the latest Harbor offline installer

wget https://github.com/goharbor/harbor/releases/download/v2.11.0/harbor-offline-installer-v2.11.0.tgz

# Extract the archive
tar xzvf harbor-offline-installer-v2.11.0.tgz

# Move to a permanent location
mv harbor /opt/harbor
cd /opt/harbor
```

## Step 2: Configure harbor.yml

```bash
# Copy the template configuration
cp harbor.yml.tmpl harbor.yml
```

Edit `/opt/harbor/harbor.yml`:

```yaml
# The hostname or IP address of your Harbor instance
hostname: registry.yourdomain.com

# HTTP configuration (redirect to HTTPS in production)
http:
  port: 80

# HTTPS configuration
https:
  port: 443
  certificate: /opt/harbor/certs/yourdomain.crt
  private_key: /opt/harbor/certs/yourdomain.key

# Initial admin password (change immediately after first login)
harbor_admin_password: Harbor12345

# Database configuration
database:
  password: root123
  max_idle_conns: 100
  max_open_conns: 900

# Default data volume
data_volume: /data/harbor

# Logging settings
log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor
```

## Step 3: Run the Harbor Installer

```bash
# Install Harbor (this generates the docker-compose.yml)
cd /opt/harbor
./install.sh --with-trivy

# Verify all services are running
docker compose ps
```

Harbor's install script generates a `docker-compose.yml` automatically. The `--with-trivy` flag enables image vulnerability scanning.

## Step 4: Import Harbor into Portainer as a Stack

Instead of using `docker compose up` from the CLI, you can manage Harbor through Portainer:

1. Open Portainer and navigate to **Stacks**
2. Click **Add Stack** → **Upload**
3. Upload the generated `/opt/harbor/docker-compose.yml`
4. Name the stack `harbor`
5. Click **Deploy the stack**

> **Note**: Harbor's generated compose file uses relative paths. You may need to adjust volume paths to absolute paths before uploading.

## Step 5: Access Harbor UI

1. Navigate to `https://registry.yourdomain.com`
2. Log in with `admin` / `Harbor12345`
3. Immediately change the admin password under **Administration** → **Users**

## Step 6: Connect Harbor to Portainer

1. In Portainer, go to **Registries** → **Add Registry**
2. Select **Custom Registry**
3. Enter:
   - **URL**: `https://registry.yourdomain.com`
   - **Username**: `admin`
   - **Password**: your Harbor admin password
4. Click **Add Registry**

## Step 7: Create a Harbor Project and Push Images

```bash
# Log in to Harbor from your workstation
docker login registry.yourdomain.com -u admin

# Tag an image for Harbor
docker tag myapp:latest registry.yourdomain.com/myproject/myapp:v1.0

# Push to Harbor
docker push registry.yourdomain.com/myproject/myapp:v1.0
```

## Step 8: Enable Vulnerability Scanning

In the Harbor UI:
1. Navigate to your project → **Configuration**
2. Enable **Automatically scan images on push**
3. Go to **Interrogation Services** → **Vulnerability** to view scan results

## Updating Harbor

```bash
# Pull new Harbor installer
# Re-run install.sh after updating harbor.yml
cd /opt/harbor
docker compose down
./install.sh --with-trivy
```

## Conclusion

Harbor provides enterprise-grade registry capabilities - vulnerability scanning, RBAC, and image signing - all manageable via Portainer stacks. This combination gives your team a powerful, self-hosted container supply chain without needing cloud registry services.
