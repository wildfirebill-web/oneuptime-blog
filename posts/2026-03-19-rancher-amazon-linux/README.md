# How to Install Rancher on Amazon Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Amazon Linux, Docker, Kubernetes, Installation

Description: A step-by-step guide to installing Rancher on Amazon Linux 2023 using Docker, including AWS-specific configuration and security group setup.

Amazon Linux 2023 is the latest generation of Amazon Linux from AWS. It is optimized for running on AWS EC2 instances and provides a stable, secure, and high-performance environment for cloud workloads. This guide covers installing Rancher on Amazon Linux 2023, including AWS-specific considerations such as security groups and instance configuration.

## Prerequisites

Before you begin, ensure you have:

- An EC2 instance running Amazon Linux 2023 with at least 4 GB RAM (t3.medium or larger)
- SSH access to the instance
- An Elastic IP or DNS name associated with the instance
- An AWS Security Group configured to allow inbound traffic on ports 80, 443, and 6443

## Step 1: Configure the Security Group

In the AWS Console, ensure your EC2 instance's security group allows the following inbound rules:

| Port  | Protocol | Source    | Purpose               |
|-------|----------|-----------|----------------------|
| 22    | TCP      | Your IP   | SSH access           |
| 80    | TCP      | 0.0.0.0/0 | HTTP                 |
| 443   | TCP      | 0.0.0.0/0 | HTTPS (Rancher UI)   |
| 6443  | TCP      | VPC CIDR  | Kubernetes API       |

## Step 2: Update the System

Connect to your instance and update all packages:

```bash
sudo dnf update -y
```

## Step 3: Install Docker

Amazon Linux 2023 provides Docker in its default repositories:

```bash
sudo dnf install -y docker
```

Enable and start Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Add your user (ec2-user) to the Docker group:

```bash
sudo usermod -aG docker ec2-user
newgrp docker
```

Verify Docker:

```bash
docker --version
docker run hello-world
```

If Docker is not available in the default repositories, install it from the Docker CE repository:

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable docker
sudo systemctl start docker
```

## Step 4: Disable Swap

Amazon Linux 2023 typically does not have swap enabled by default on EC2 instances. Verify:

```bash
free -h
```

If swap is enabled, disable it:

```bash
sudo swapoff -a
```

## Step 5: Configure Kernel Modules

Load the required kernel modules:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/rancher.conf
br_netfilter
overlay
EOF

sudo modprobe br_netfilter
sudo modprobe overlay
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

## Step 6: Configure Docker Logging

```bash
sudo mkdir -p /etc/docker

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

## Step 7: Set Up Persistent Storage

For EC2 instances, consider using an EBS volume for Rancher data to ensure durability:

```bash
# If using the root volume

sudo mkdir -p /opt/rancher

# If using a separate EBS volume (recommended for production)
# Attach an EBS volume in the AWS Console, then:
# sudo mkfs.xfs /dev/xvdf
# sudo mkdir -p /opt/rancher
# sudo mount /dev/xvdf /opt/rancher
# echo '/dev/xvdf /opt/rancher xfs defaults 0 0' | sudo tee -a /etc/fstab
```

## Step 8: Run Rancher

Deploy Rancher:

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

## Step 9: Get the Bootstrap Password

```bash
docker logs rancher 2>&1 | grep "Bootstrap Password:"
```

## Step 10: Access the Rancher UI

Navigate to `https://<your-elastic-ip>` or `https://<your-dns-name>` in your browser. Accept the self-signed certificate warning and enter the bootstrap password.

Complete the initial setup:

1. Set a new admin password
2. Set the Rancher server URL to your Elastic IP or DNS name
3. Accept the terms and conditions

## AWS-Specific Considerations

**Instance metadata**: Amazon Linux 2023 uses IMDSv2 by default. Rancher can leverage AWS cloud provider integration when provisioning clusters:

```bash
# Verify IMDSv2 is working
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

**IAM roles**: If you plan to use Rancher to provision EKS clusters or manage AWS resources, attach an IAM role to your EC2 instance with the appropriate permissions.

**Elastic IP**: Always use an Elastic IP for your Rancher instance to ensure the IP does not change after a stop/start cycle.

**Backup with EBS snapshots**: Take regular EBS snapshots for backup:

```bash
# Get the instance ID
INSTANCE_ID=$(TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600") && curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
echo "Instance ID: $INSTANCE_ID"
```

You can then create EBS snapshots via the AWS Console or CLI.

## Troubleshooting

```bash
# Check Docker service
sudo systemctl status docker

# View Rancher logs
docker logs rancher --tail 100

# Check security group (from instance)
curl -s http://checkip.amazonaws.com
# Verify the returned IP can access ports 80 and 443

# Check system resources
free -h
df -h

# Check DNS resolution
nslookup rancher.yourdomain.com
```

## Conclusion

You have successfully installed Rancher on Amazon Linux 2023. Running Rancher on Amazon Linux within AWS gives you tight integration with AWS services, making it easier to provision and manage EKS clusters and other AWS resources directly from the Rancher dashboard. The combination of Amazon Linux's optimized performance on EC2 and Rancher's multi-cluster management capabilities provides a strong foundation for your Kubernetes operations.
