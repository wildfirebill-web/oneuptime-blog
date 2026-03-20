# How to Deploy Portainer on AWS EC2

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, AWS, EC2, Docker, Cloud, Self-Hosted, DevOps

Description: Deploy Portainer on an AWS EC2 instance with proper security group configuration, IAM roles, and optional EBS volume for production-ready container management.

## Introduction

Running Portainer on AWS EC2 gives you a scalable container management platform in the cloud. With proper security group configuration and optional integration with AWS services like ECR, you can manage Docker containers on EC2 with the same interface you use locally. This guide covers deployment, security, and AWS-specific integrations.

## Prerequisites

- AWS account with EC2 access
- AWS CLI configured (optional but helpful)
- Basic AWS EC2 knowledge

## Step 1: Launch an EC2 Instance

### Recommended Instance

For Portainer and a small number of managed containers:
- **Instance type**: `t3.medium` (2 vCPU, 4GB RAM) minimum
- **AMI**: Ubuntu Server 24.04 LTS (64-bit)
- **Storage**: 20GB GP3 EBS volume (root) + optional 50GB data volume
- **VPC**: Use your existing VPC or create a new one

### Launch via AWS Console

1. Navigate to **EC2 > Launch Instance**
2. Select **Ubuntu Server 24.04 LTS**
3. Choose instance type `t3.medium`
4. Configure network: select your VPC and public subnet
5. Enable **Auto-assign public IP** (or use Elastic IP for production)
6. Set storage: 20GB GP3 root volume
7. Create or select a key pair

## Step 2: Configure Security Group

Create a security group named `portainer-sg`:

| Type | Protocol | Port | Source |
|------|---------|------|--------|
| SSH | TCP | 22 | Your IP |
| Custom TCP | TCP | 9000 | Your IP (or VPN range) |
| Custom TCP | TCP | 9443 | Your IP (or VPN range) |
| HTTP | TCP | 80 | 0.0.0.0/0 (if running public services) |
| HTTPS | TCP | 443 | 0.0.0.0/0 (if running public services) |

**Important**: Never open Portainer ports to `0.0.0.0/0` in production. Restrict access to your IP or VPN.

### Via AWS CLI

```bash
# Create security group
aws ec2 create-security-group \
    --group-name portainer-sg \
    --description "Portainer container management" \
    --vpc-id vpc-xxxxxxxx

# Get your public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)

# Allow Portainer ports from your IP only
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxxxx \
    --protocol tcp --port 22 --cidr ${MY_IP}/32

aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxxxx \
    --protocol tcp --port 9000 --cidr ${MY_IP}/32

aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxxxx \
    --protocol tcp --port 9443 --cidr ${MY_IP}/32
```

## Step 3: Connect and Install Docker

```bash
# SSH to EC2 instance
ssh -i ~/.ssh/your-key.pem ubuntu@<ec2-public-ip>

# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
newgrp docker

sudo systemctl enable docker
```

## Step 4: Attach and Mount Data Volume (Optional)

For production, use a separate EBS volume for Docker data:

```bash
# Check available block devices
lsblk

# Format the data volume (e.g., /dev/xvdf)
sudo mkfs.ext4 /dev/xvdf

# Mount it
sudo mkdir -p /data
echo '/dev/xvdf /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
sudo mount -a

# Point Docker to the new volume
sudo mkdir -p /data/docker
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "data-root": "/data/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
sudo systemctl restart docker
```

## Step 5: Deploy Portainer

```bash
# Create data volume
docker volume create portainer_data

# Deploy Portainer
docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 6: Configure EC2 Instance Connect or Systems Manager

For more secure access without open SSH port:

```bash
# Install SSM agent (usually pre-installed on Ubuntu AMIs)
sudo snap install amazon-ssm-agent --classic
sudo systemctl enable --now snap.amazon-ssm-agent.amazon-ssm-agent

# Attach IAM role with AmazonSSMManagedInstanceCore policy to EC2
```

## Step 7: Integrate with Amazon ECR

In Portainer, add your ECR registry:

1. Navigate to **Registries > Add registry**
2. Select **Custom registry**
3. URL: `<account-id>.dkr.ecr.<region>.amazonaws.com`
4. For authentication, use the ECR token:

```bash
# Get ECR login token (valid for 12 hours)
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin \
    <account-id>.dkr.ecr.us-east-1.amazonaws.com
```

## Step 8: Assign an Elastic IP

For production, assign an Elastic IP so the address doesn't change on restart:

```bash
# Allocate Elastic IP
aws ec2 allocate-address --domain vpc

# Associate with EC2 instance
aws ec2 associate-address \
    --instance-id i-xxxxxxxxxxxxxxxxx \
    --allocation-id eipalloc-xxxxxxxxxxxxxxxxx
```

## Conclusion

Portainer on AWS EC2 provides a familiar container management interface for cloud-based workloads. By restricting security group access to specific IPs and using EBS volumes for data persistence, you create a secure and reliable setup. The AWS ECR integration allows you to deploy private container images directly from the Portainer UI.
