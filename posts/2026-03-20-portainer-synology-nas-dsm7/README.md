# How to Install Portainer on Synology NAS (DSM 7)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Synology, NAS, Docker, DSM 7, Self-Hosted, Home Lab

Description: Install Portainer on a Synology NAS running DSM 7 to manage Docker containers through a web UI instead of the limited Container Manager interface.

## Introduction

Synology's DSM 7 includes Container Manager (formerly Docker) for running containers, but its interface is limited compared to Portainer. Installing Portainer on your Synology NAS gives you stack support, registry management, environment templates, and a much more powerful container management experience.

## Prerequisites

- Synology NAS with DSM 7.x installed
- Container Manager package installed from Package Center
- SSH access enabled (Control Panel > Terminal & SNMP)
- At least 1GB free RAM and 2GB free disk space

## Installation Methods

### Method 1: Via Container Manager UI (Recommended)

1. Open **Container Manager** on your Synology
2. Navigate to **Registry** and search for `portainer`
3. Select `portainer/portainer-ce` and click **Download**
4. Choose tag `latest` and click **Apply**
5. Navigate to **Container** and click **Create**
6. Select the `portainer/portainer-ce` image
7. Configure:
   - **Container Name**: `portainer`
   - **Enable auto-restart**: Yes
8. Click **Advanced Settings**
9. Under **Volume**, add:
   - Host path: `/var/run/docker.sock` → Mount path: `/var/run/docker.sock` (Type: File)
   - Docker volume: `portainer_data` → Mount path: `/data`
10. Under **Port Settings**, add:
    - Local port: `9000` → Container port: `9000` (TCP)
    - Local port: `9443` → Container port: `9443` (TCP)
11. Click **Apply** and **Next**, then **Done**

### Method 2: Via SSH (Recommended for Reproducibility)

SSH into your Synology NAS:

```bash
ssh admin@<synology-ip>
```

Run the Portainer container:

```bash
# Create a volume for Portainer data persistence
sudo docker volume create portainer_data

# Run Portainer CE
sudo docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Verify it is running
sudo docker ps | grep portainer
```

### Step 3: Access Portainer

Open your browser and navigate to:
- HTTP: `http://<synology-ip>:9000`
- HTTPS: `https://<synology-ip>:9443`

On first access, you'll be prompted to create an admin user.

## Configuring DSM Firewall

If the Synology firewall is enabled, allow the Portainer ports:

1. Go to **Control Panel > Security > Firewall**
2. Click **Edit Rules** for your profile
3. Click **Create** and set:
   - Ports: `9000`, `9443`
   - Protocol: TCP
   - Source IP: your local subnet (e.g., `192.168.1.0/24`)
   - Action: Allow
4. Move the rule above the default deny rule

## Post-Installation Configuration

### Enable HTTPS with a Custom Certificate

Synology can provide its SSL certificate to Portainer. Copy the certificate files:

```bash
# Synology stores certificates in /usr/syno/etc/certificate/
# Find your default certificate
ls /usr/syno/etc/certificate/_archive/

# Copy to Portainer (adjust path as needed)
sudo docker cp /usr/syno/etc/certificate/_archive/<cert-id>/fullchain.pem portainer:/certs/cert.pem
sudo docker cp /usr/syno/etc/certificate/_archive/<cert-id>/privkey.pem portainer:/certs/key.pem
```

Then recreate the container with SSL certificate mounts:

```bash
sudo docker stop portainer && sudo docker rm portainer

sudo docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /path/to/certs:/certs \
  portainer/portainer-ce:latest \
  --sslcert /certs/cert.pem \
  --sslkey /certs/key.pem
```

## Updating Portainer

```bash
# Stop and remove the old container (data volume is preserved)
sudo docker stop portainer
sudo docker rm portainer

# Pull the latest image
sudo docker pull portainer/portainer-ce:latest

# Recreate with the same run command
sudo docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Troubleshooting

**Cannot access port 9000:** Check DSM firewall settings and verify the container is running with `sudo docker ps`.

**Permission denied on docker.sock:** Ensure you are running the docker command with `sudo` on Synology.

**Container restarts in a loop:** Check logs with `sudo docker logs portainer`.

## Conclusion

Portainer on Synology DSM 7 transforms your NAS into a capable container management platform. With Portainer's stack support, you can deploy complex multi-container applications on your NAS that would be difficult or impossible through Container Manager alone. The persistent data volume ensures your configuration survives container updates.
