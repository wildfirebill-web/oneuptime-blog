# How to Install Portainer CE on Amazon Linux with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, amazon-linux, aws, docker, installation

Description: A guide to installing Portainer Community Edition on Amazon Linux 2 and Amazon Linux 2023 with Docker for AWS EC2 deployments.

## Overview

Amazon Linux is the default EC2 instance OS for many AWS users. This guide covers installing Docker and Portainer CE on both Amazon Linux 2 and Amazon Linux 2023, which are optimized for AWS environments.

## Prerequisites

- Amazon Linux 2 or Amazon Linux 2023 EC2 instance
- t3.small or larger (minimum 2GB RAM)
- Security group with inbound ports 9443 and 8000 open (from your IP)
- SSM Session Manager or SSH access

## EC2 Security Group Configuration

Before installation, configure your EC2 security group to allow Portainer access:

```
AWS Console → EC2 → Security Groups → Your SG → Inbound Rules

Add rules:
- Type: Custom TCP, Port: 9443, Source: Your IP/32 (or 0.0.0.0/0 for testing only)
- Type: Custom TCP, Port: 8000, Source: Your IP/32
```

## Step 1: Update System

```bash
# Amazon Linux 2
sudo yum update -y

# Amazon Linux 2023
sudo dnf update -y
```

## Step 2: Install Docker

### Amazon Linux 2

```bash
# Install Docker from Amazon's extras library
sudo amazon-linux-extras install docker -y

# Start and enable Docker
sudo systemctl enable --now docker

# Add ec2-user to docker group
sudo usermod -aG docker ec2-user
newgrp docker
```

### Amazon Linux 2023

```bash
# Docker is available directly in AL2023
sudo dnf install -y docker

# Start and enable Docker
sudo systemctl enable --now docker

# Add ec2-user to docker group
sudo usermod -aG docker ec2-user
newgrp docker
```

## Step 3: Verify Docker

```bash
# Verify installation
docker --version
docker run hello-world
```

## Step 4: Deploy Portainer CE

```bash
# Create data volume
docker volume create portainer_data

# Deploy Portainer CE
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Verify deployment
docker ps | grep portainer
```

## Step 5: Access Portainer via EC2

```bash
# Get the EC2 public IP
INSTANCE_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)
echo "Portainer URL: https://${INSTANCE_IP}:9443"
```

Navigate to the URL in your browser and complete the initial setup.

## Optional: Use AWS Application Load Balancer

For production deployments, route traffic through an ALB:

```
ALB Listener (HTTPS/443) → Target Group (HTTP/9000) → EC2 Instance
```

```bash
# Start Portainer with HTTP port for ALB health checks
docker stop portainer && docker rm portainer
docker run -d \
  -p 8000:8000 \
  -p 9000:9000 \    # HTTP port for ALB
  -p 9443:9443 \    # HTTPS direct
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --http-enabled    # Enable HTTP
```

## Monitoring Portainer with CloudWatch

```bash
# Install CloudWatch agent
sudo yum install -y amazon-cloudwatch-agent

# Configure Portainer log forwarding to CloudWatch
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/lib/docker/containers/*/*-json.log",
            "log_group_name": "portainer",
            "log_stream_name": "{instance_id}/portainer"
          }
        ]
      }
    }
  }
}
EOF

sudo systemctl start amazon-cloudwatch-agent
```

## Conclusion

Installing Portainer CE on Amazon Linux is straightforward with the additional context of AWS security groups and EC2 instance metadata. The Amazon Linux 2023 package manager makes Docker installation even simpler. For production AWS deployments, pair Portainer with an Application Load Balancer for SSL termination and high availability, and use CloudWatch for log aggregation.
