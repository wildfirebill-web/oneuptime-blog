# How to Deploy Portainer on Oracle Cloud Free Tier

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Oracle Cloud, Free Tier, Docker, ARM

Description: Learn how to deploy Portainer on Oracle Cloud Infrastructure's Always Free tier, including the ARM-based Ampere A1 instances that provide up to 4 vCPUs and 24GB RAM for free.

## Oracle Cloud Free Tier Resources

Oracle's Always Free tier includes:
- 2 AMD-based micro VMs (1/8 OCPU, 1GB RAM each)
- **Up to 4 Ampere A1 cores and 24GB RAM** (ARM-based, significantly more powerful)
- 200GB block storage total

The A1 Ampere instances are excellent for Portainer — 4 cores and 24GB RAM is more than most cloud free tiers offer.

## Prerequisites

- Oracle Cloud account (requires credit card for verification, not charged)
- OCI CLI configured, or use the OCI Console

## Step 1: Create an ARM A1 Compute Instance

In the OCI Console:

1. **Compute → Instances → Create Instance**
2. **Image**: Canonical Ubuntu 22.04 (minimal)
3. **Shape**: Change to Ampere → A1.Flex
   - OCPUs: 4
   - Memory: 24GB
4. Add your SSH public key
5. Under **Networking**: ensure public IP is assigned

## Step 2: Configure Security List

OCI uses Security Lists for firewall rules (VCN-level):

1. **Networking → Virtual Cloud Networks → your-vcn → Security Lists → Default**
2. Add Ingress Rules:

```
Protocol: TCP, Source: 0.0.0.0/0, Port: 22     (SSH)
Protocol: TCP, Source: 0.0.0.0/0, Port: 9443   (Portainer HTTPS)
Protocol: TCP, Source: 0.0.0.0/0, Port: 80     (HTTP)
Protocol: TCP, Source: 0.0.0.0/0, Port: 443    (HTTPS)
```

## Step 3: Configure iptables on the Instance

OCI Ubuntu instances have an additional iptables firewall:

```bash
# SSH into the instance
ssh ubuntu@<PUBLIC_IP>

# Allow required ports in iptables
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 9443 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT

# Save rules
sudo netfilter-persistent save
```

## Step 4: Install Docker and Portainer

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker (ARM64 native)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
newgrp docker

# Verify Docker architecture
docker version --format '{{.Server.Arch}}'
# Expected: arm64

# Install Portainer
docker volume create portainer_data

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Attach Block Volume for Persistent Storage

For larger Docker deployments, use a dedicated block volume:

1. **Storage → Block Volumes → Create Block Volume** (50GB, free tier)
2. **Attach** to your instance
3. On the instance:

```bash
# Find the volume
ls /dev/oracleoci/

# Format and mount
sudo mkfs.ext4 /dev/oracleoci/oraclevdb
sudo mkdir /mnt/docker-data
sudo mount /dev/oracleoci/oraclevdb /mnt/docker-data

# Move Docker data root
sudo systemctl stop docker
sudo rsync -avz /var/lib/docker/ /mnt/docker-data/
echo '{"data-root": "/mnt/docker-data"}' | sudo tee /etc/docker/daemon.json
sudo systemctl start docker
```

## What You Can Run for Free

With 4 ARM cores and 24GB RAM on OCI Free Tier:

- Portainer (management UI)
- Multiple application stacks
- Nginx Proxy Manager (reverse proxy)
- A database (PostgreSQL, MySQL)
- Monitoring stack (Prometheus + Grafana)
- A Kubernetes cluster with k3s

## Conclusion

Oracle Cloud Free Tier's A1 Ampere instances are the most generous free cloud compute available for running Portainer. The ARM architecture is fully supported by Portainer and most Docker images. The main friction point is OCI's two-layer firewall (Security List + iptables) — remember to configure both.
