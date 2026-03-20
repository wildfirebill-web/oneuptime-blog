# How to Install Rancher on openSUSE

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, openSUSE, Docker, Kubernetes, Installation

Description: A step-by-step guide to installing Rancher on openSUSE Leap using Docker, including zypper package management and system configuration.

openSUSE is the community distribution from SUSE and has a natural affinity with Rancher since SUSE acquired Rancher Labs. openSUSE Leap provides a stable, enterprise-aligned distribution that shares its codebase with SUSE Linux Enterprise Server. This guide covers installing Rancher on openSUSE Leap 15.5 or later using Docker.

## Prerequisites

Before you begin, ensure you have:

- A server running openSUSE Leap 15.5 or later with at least 4 GB RAM and 2 CPU cores
- Root or sudo access
- A static IP address or DNS name
- Internet access for downloading packages

## Step 1: Update the System

Update all packages using zypper:

```bash
sudo zypper refresh
sudo zypper update -y
```

## Step 2: Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## Step 3: Install Required Dependencies

```bash
sudo zypper install -y \
  curl \
  wget \
  tar \
  gzip \
  iptables
```

## Step 4: Install Docker

openSUSE provides Docker in its official repositories:

```bash
sudo zypper install -y docker docker-compose
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

## Step 5: Configure the Firewall

openSUSE uses `firewalld` by default. Open the required ports:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --reload
```

Verify the rules:

```bash
sudo firewall-cmd --list-all
```

## Step 6: Configure Kernel Modules

Load the required modules:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/rancher.conf
br_netfilter
overlay
EOF

sudo modprobe br_netfilter
sudo modprobe overlay
```

Set sysctl parameters:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-rancher.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## Step 7: Configure AppArmor

openSUSE uses AppArmor by default. Check its status:

```bash
sudo systemctl status apparmor
sudo aa-status
```

If you encounter issues with Rancher containers, you can set AppArmor to complain mode for Docker:

```bash
sudo aa-complain /etc/apparmor.d/usr.sbin.docker*
```

## Step 8: Configure Docker Logging

```bash
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl restart docker
```

## Step 9: Create Persistent Storage

```bash
sudo mkdir -p /opt/rancher
```

## Step 10: Run Rancher

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

## Step 11: Get the Bootstrap Password

Wait about a minute, then:

```bash
docker logs rancher 2>&1 | grep "Bootstrap Password:"
```

## Step 12: Access the Rancher UI

Navigate to `https://<your-server-ip>` in your browser. Accept the self-signed certificate warning and enter the bootstrap password.

Complete the setup:

1. Set a new admin password
2. Configure the Rancher server URL
3. Accept the terms and conditions

## openSUSE Specific Considerations

**SUSE ecosystem integration**: Since Rancher is a SUSE product, openSUSE provides excellent compatibility. You can also use RKE2 (Rancher Kubernetes Engine) for provisioning downstream clusters, which is SUSE's hardened Kubernetes distribution.

**Btrfs file system**: openSUSE often uses Btrfs as the default file system. Docker works well with Btrfs, but you may want to use overlay2 instead for better performance:

```bash
# Check current filesystem

df -T /

# The daemon.json already specifies overlay2 as the storage driver
```

**YaST integration**: You can use YaST for some configuration tasks:

```bash
# Open firewall configuration
sudo yast2 firewall

# Open network configuration
sudo yast2 lan
```

**Transactional updates**: If using openSUSE MicroOS (a variant designed for containers), use transactional-update instead of zypper:

```bash
# Only for MicroOS
# sudo transactional-update pkg install docker
```

## Updating Rancher

To update Rancher to a newer version:

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

## Troubleshooting

```bash
# Check Docker service
sudo systemctl status docker
sudo journalctl -u docker --tail 50

# View Rancher logs
docker logs rancher --tail 100

# Check AppArmor denials
sudo dmesg | grep -i apparmor

# Check firewall
sudo firewall-cmd --list-all

# Check resources
free -h
df -h
```

## Conclusion

You have successfully installed Rancher on openSUSE Leap. Given that Rancher is part of the SUSE family, openSUSE provides one of the most natural platforms for running Rancher. The tight integration between the SUSE ecosystem and Rancher ensures smooth operation and excellent support for your Kubernetes management needs.
