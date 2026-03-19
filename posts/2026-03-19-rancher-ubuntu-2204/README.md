# How to Install Rancher on Ubuntu 22.04

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Ubuntu, Docker, Kubernetes, Installation

Description: A step-by-step guide to installing Rancher on Ubuntu 22.04 LTS using Docker, covering system preparation, Docker installation, and Rancher deployment.

Ubuntu 22.04 LTS (Jammy Jellyfish) is one of the most popular Linux distributions for server deployments. Its long-term support lifecycle and wide package availability make it an excellent choice for running Rancher. This guide walks you through installing Rancher on Ubuntu 22.04 from a fresh server setup to a fully functional Rancher deployment.

## Prerequisites

Before you begin, ensure you have:

- A server running Ubuntu 22.04 LTS with at least 4 GB RAM and 2 CPU cores
- Root or sudo access
- A static IP address or DNS name for your server
- Ports 80 and 443 open in your firewall

## Step 1: Update the System

Start by updating all packages to their latest versions:

```bash
sudo apt update && sudo apt upgrade -y
```

## Step 2: Install Required Dependencies

Install the packages that Docker and Rancher need:

```bash
sudo apt install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  software-properties-common
```

## Step 3: Install Docker

Add the official Docker GPG key and repository:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker Engine:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Start and enable Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Add your user to the Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker is running:

```bash
docker --version
docker run hello-world
```

## Step 4: Configure Firewall

If you are using UFW (Ubuntu's default firewall), allow the necessary ports:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 6443/tcp
sudo ufw reload
```

Port 6443 is used by the Kubernetes API server when Rancher provisions clusters.

## Step 5: Configure System Settings

Ensure the required kernel modules are loaded:

```bash
sudo modprobe br_netfilter
sudo modprobe overlay

cat <<EOF | sudo tee /etc/modules-load.d/rancher.conf
br_netfilter
overlay
EOF
```

Set the required sysctl parameters:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-rancher.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## Step 6: Create Persistent Storage

Create a directory to store Rancher data:

```bash
sudo mkdir -p /opt/rancher
```

## Step 7: Run Rancher

Deploy Rancher using Docker:

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

## Step 8: Retrieve the Bootstrap Password

Wait about 60 seconds for Rancher to initialize, then retrieve the bootstrap password:

```bash
docker logs rancher 2>&1 | grep "Bootstrap Password:"
```

## Step 9: Access the Rancher UI

Open your browser and navigate to `https://<your-server-ip>`. Accept the self-signed certificate warning and enter the bootstrap password.

Complete the initial setup by:

1. Setting a new admin password
2. Configuring the Rancher server URL
3. Accepting the terms and conditions

## Step 10: Configure Log Rotation

To prevent Docker logs from consuming too much disk space, configure log rotation:

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

Note that restarting Docker will restart the Rancher container as well.

## Setting Up Automatic Updates

Create a script to check for security updates automatically:

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

## Troubleshooting

If you encounter issues, check the following:

```bash
# Check Docker status
sudo systemctl status docker

# Check Rancher container logs
docker logs rancher --tail 100

# Check system resources
free -h
df -h

# Check if ports are in use
sudo ss -tlnp | grep -E ':(80|443)'
```

Common issues on Ubuntu 22.04:

- **AppArmor conflicts**: If Rancher fails to start, try running `sudo aa-remove-unknown`
- **DNS resolution**: Check `/etc/resolv.conf` has valid nameservers
- **Swap**: Kubernetes prefers swap to be disabled. Run `sudo swapoff -a` and remove the swap entry from `/etc/fstab`

## Conclusion

You have successfully installed Rancher on Ubuntu 22.04 LTS. Your Rancher instance is now ready to create and manage Kubernetes clusters. Ubuntu 22.04 will receive security updates until April 2027, giving you a stable foundation for your container management platform.
