# How to Install Rancher on a Single Node with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Docker, Kubernetes, Installation

Description: A step-by-step guide to installing Rancher Server on a single node using Docker for quick development and testing environments.

Rancher is a powerful open-source platform for managing Kubernetes clusters. One of the fastest ways to get Rancher up and running is by deploying it as a Docker container on a single node. This approach is ideal for development, testing, and small-scale production environments where high availability is not a strict requirement.

In this guide, you will learn how to install Rancher on a single node using Docker, configure persistent storage, and access the Rancher UI.

## Prerequisites

Before you begin, make sure you have:

- A Linux server (Ubuntu, CentOS, Debian, or similar) with at least 4 GB of RAM and 2 CPU cores
- Docker installed (version 20.10 or later)
- A domain name or IP address pointing to your server
- Ports 80 and 443 open on your firewall

## Step 1: Install Docker

If Docker is not already installed, run the following commands to install it:

```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl enable docker
sudo systemctl start docker
```

Verify the installation:

```bash
docker --version
```

Add your user to the Docker group so you can run Docker commands without sudo:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## Step 2: Create a Persistent Volume for Rancher Data

Rancher stores its data including cluster configurations, user accounts, and settings. To ensure this data persists across container restarts, create a directory on the host:

```bash
sudo mkdir -p /opt/rancher
```

## Step 3: Run the Rancher Container

Deploy Rancher using Docker with the following command:

```bash
docker run -d \
  --name rancher \
  --restart=unless-stopped \
  -p 80:80 \
  -p 443:443 \
  -v /opt/rancher:/var/lib/rancher \
  --privileged \
  rancher/rancher:latest
```

Here is what each flag does:

- `-d` runs the container in the background
- `--restart=unless-stopped` ensures the container restarts automatically after a reboot
- `-p 80:80 -p 443:443` maps HTTP and HTTPS ports
- `-v /opt/rancher:/var/lib/rancher` mounts persistent storage
- `--privileged` grants the container the necessary permissions to manage Kubernetes

## Step 4: Check the Container Status

Wait about a minute for Rancher to initialize, then check the container status:

```bash
docker ps
docker logs rancher 2>&1 | tail -20
```

You should see the Rancher container running and logs indicating that the server is starting up.

## Step 5: Retrieve the Bootstrap Password

Rancher generates a bootstrap password on first startup. Retrieve it with:

```bash
docker logs rancher 2>&1 | grep "Bootstrap Password:"
```

If the password does not appear in the logs yet, wait a few more seconds and try again. You can also set a specific bootstrap password at startup:

```bash
docker run -d \
  --name rancher \
  --restart=unless-stopped \
  -p 80:80 \
  -p 443:443 \
  -v /opt/rancher:/var/lib/rancher \
  --privileged \
  -e CATTLE_BOOTSTRAP_PASSWORD=yourSecurePassword \
  rancher/rancher:latest
```

## Step 6: Access the Rancher UI

Open your web browser and navigate to:

```plaintext
https://<your-server-ip>
```

You will see a self-signed certificate warning. Accept the warning to proceed. On the first login page, enter the bootstrap password you retrieved in the previous step.

You will then be prompted to:

1. Set a new admin password
2. Configure the Rancher server URL
3. Agree to the terms and conditions

## Step 7: Configure the Server URL

Set the server URL to the address that your downstream clusters will use to communicate with Rancher. This should be a DNS name or IP address that is reachable from all nodes:

```plaintext
https://rancher.yourdomain.com
```

## Step 8: Verify the Installation

After logging in, you should see the Rancher dashboard. From here you can:

- Create new Kubernetes clusters
- Import existing clusters
- Deploy workloads and manage resources

Verify Rancher is healthy by checking the container logs:

```bash
docker logs rancher 2>&1 | grep -i "ready"
```

## Backing Up Rancher Data

Since all Rancher data is stored in `/opt/rancher`, you can back it up by stopping the container and copying the directory:

```bash
docker stop rancher
tar czf rancher-backup-$(date +%Y%m%d).tar.gz /opt/rancher
docker start rancher
```

## Upgrading Rancher

To upgrade Rancher to a newer version:

```bash
docker stop rancher
docker rm rancher
docker pull rancher/rancher:latest

docker run -d \
  --name rancher \
  --restart=unless-stopped \
  -p 80:80 \
  -p 443:443 \
  -v /opt/rancher:/var/lib/rancher \
  --privileged \
  rancher/rancher:latest
```

Your data will be preserved because it is stored in the mounted volume.

## Troubleshooting

If Rancher does not start properly, check the following:

- **Port conflicts**: Ensure no other service is using ports 80 or 443
- **Insufficient resources**: Rancher needs at least 4 GB of RAM
- **Docker version**: Make sure Docker is version 20.10 or later
- **Firewall rules**: Ensure ports 80 and 443 are open

View detailed logs:

```bash
docker logs rancher --tail 100
```

## Conclusion

You have successfully installed Rancher on a single node using Docker. This setup provides a quick way to get started with Rancher for managing Kubernetes clusters. For production environments that require high availability, consider deploying Rancher on a multi-node Kubernetes cluster using Helm.
