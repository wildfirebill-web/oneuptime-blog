# How to Install Rancher on Rocky Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Rocky Linux, Docker, Kubernetes, Installation

Description: A step-by-step guide to installing Rancher on Rocky Linux 9 using Docker, covering system preparation, Docker setup, firewall configuration, and deployment.

Rocky Linux was created as a direct replacement for CentOS Linux after Red Hat shifted CentOS to a rolling release model. It maintains full binary compatibility with RHEL, making it an excellent choice for enterprise workloads including Rancher. This guide covers the complete process of installing Rancher on Rocky Linux 9.

## Prerequisites

Before you begin, ensure you have:

- A server running Rocky Linux 9 with at least 4 GB RAM and 2 CPU cores
- Root or sudo access
- A static IP address or DNS name
- Internet access for downloading packages and container images

## Step 1: Update the System

Start by updating all packages:

```bash
sudo dnf update -y
```

Reboot if kernel updates were applied:

```bash
sudo reboot
```

## Step 2: Disable Swap

Disable swap for Kubernetes compatibility:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## Step 3: Configure SELinux

Set SELinux to permissive mode for Rancher:

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

## Step 4: Install Required Dependencies

```bash
sudo dnf install -y \
  yum-utils \
  device-mapper-persistent-data \
  lvm2 \
  curl \
  wget \
  tar
```

## Step 5: Remove Podman and Buildah

Rocky Linux ships with Podman by default. Remove it to avoid conflicts with Docker:

```bash
sudo dnf remove -y podman buildah containers-common
```

## Step 6: Install Docker

Add the Docker repository:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

Install Docker Engine:

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Enable and start Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Add your user to the Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker:

```bash
docker --version
docker run hello-world
```

## Step 7: Configure the Firewall

Open the required ports using firewalld:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --reload
```

## Step 8: Configure Kernel Modules

Load and persist the required kernel modules:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/rancher.conf
br_netfilter
overlay
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF

sudo modprobe br_netfilter
sudo modprobe overlay
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh
```

Configure sysctl parameters:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-rancher.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## Step 9: Configure Docker Logging

Set up log rotation to prevent excessive disk usage:

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

## Step 10: Create Persistent Storage

```bash
sudo mkdir -p /opt/rancher
```

## Step 11: Run Rancher

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

## Step 12: Retrieve the Bootstrap Password

```bash
docker logs rancher 2>&1 | grep "Bootstrap Password:"
```

If the password has not appeared yet, wait another minute and try again.

## Step 13: Access the Rancher UI

Navigate to `https://<your-server-ip>` in your browser. Accept the self-signed certificate warning and enter the bootstrap password.

Complete the setup:

1. Set a new admin password
2. Configure the Rancher server URL
3. Accept the terms and conditions

## Rocky Linux Specific Considerations

**FIPS mode**: Rocky Linux supports FIPS 140-2 mode. If FIPS is enabled on your server, ensure you are using a Rancher version that supports FIPS compliance.

**Cockpit**: Rocky Linux often comes with Cockpit installed on port 9090. This does not conflict with Rancher, but be aware of it when managing your server.

**Package compatibility**: Since Rocky Linux is binary compatible with RHEL, you can use the CentOS Docker repository without issues:

```bash
cat /etc/os-release | grep -i rocky
```

## Backup and Recovery

Create regular backups:

```bash
docker stop rancher
sudo tar czf /backup/rancher-backup-$(date +%Y%m%d).tar.gz /opt/rancher
docker start rancher
```

## Troubleshooting

```bash
# Check Docker service

sudo systemctl status docker

# View Rancher container logs
docker logs rancher --tail 100

# Check SELinux denials
sudo ausearch -m avc -ts recent

# Check firewall configuration
sudo firewall-cmd --list-all

# Monitor resources
top -bn1 | head -20
free -h
```

## Conclusion

You have successfully installed Rancher on Rocky Linux 9. Rocky Linux provides an enterprise-grade, RHEL-compatible platform that is well suited for running Rancher and managing Kubernetes clusters. With its community-driven development and long support lifecycle, Rocky Linux is a reliable foundation for your container management infrastructure.
