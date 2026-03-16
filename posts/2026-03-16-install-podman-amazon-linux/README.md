# How to Install Podman on Amazon Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Installation, Linux, Containers, DevOps, Amazon Linux

Description: Step-by-step instructions for installing Podman on Amazon Linux 2 and Amazon Linux 2023 for running containers on AWS EC2 instances.

---

> Amazon Linux 2023 includes Podman as the default container runtime, replacing Docker as the recommended container tool on AWS.

Amazon Linux is the go-to operating system for AWS EC2 instances. Amazon Linux 2023 (AL2023) ships with Podman in its default repositories, while Amazon Linux 2 requires the `extras` library. This guide covers both versions.

---

## Prerequisites

- An Amazon Linux 2 or Amazon Linux 2023 EC2 instance
- SSH access with a user that has sudo privileges
- An active internet connection

## Installing on Amazon Linux 2023

### Step 1: Update the System

```bash
# Update all packages
sudo dnf update -y
```

### Step 2: Install Podman

Podman is available directly from the AL2023 repositories:

```bash
# Install Podman
sudo dnf install -y podman
```

### Step 3: Verify the Installation

```bash
# Check the version
podman --version

# Display system information
podman info
```

## Installing on Amazon Linux 2

### Step 1: Update the System

```bash
# Update all packages
sudo yum update -y
```

### Step 2: Enable the Extras Repository

Podman is available through the Amazon Linux 2 extras library:

```bash
# List available extras topics
amazon-linux-extras list | grep -i container

# Enable and install the container tools
sudo amazon-linux-extras enable docker
sudo yum install -y podman
```

If the above does not work, try the `container-tools` extra:

```bash
# Alternative: use container-tools extras topic
sudo amazon-linux-extras install -y podman
```

### Step 3: Verify the Installation

```bash
# Check the version
podman --version
```

## Configure Rootless Containers (Both Versions)

### Step 4: Set Up User Namespaces

```bash
# Enable user namespaces (Amazon Linux 2)
echo "user.max_user_namespaces=28633" | sudo tee -a /etc/sysctl.d/userns.conf
sudo sysctl -p /etc/sysctl.d/userns.conf

# Add subuid and subgid mappings for ec2-user
sudo usermod --add-subuids 100000-165535 ec2-user
sudo usermod --add-subgids 100000-165535 ec2-user

# Verify the mappings
cat /etc/subuid
cat /etc/subgid
```

### Step 5: Install Rootless Dependencies

```bash
# Install slirp4netns for rootless networking
sudo dnf install -y slirp4netns   # AL2023
# or
sudo yum install -y slirp4netns   # AL2
```

## Step 6: Run a Test Container

```bash
# Run a test container
podman run --rm docker.io/library/hello-world
```

## Step 7: Enable the Podman Socket

```bash
# Enable the Podman socket for Docker compatibility
systemctl --user enable --now podman.socket

# Verify it is running
systemctl --user status podman.socket
```

## Running a Practical Example

Deploy a containerized application on your EC2 instance:

```bash
# Run an Nginx container
podman run -d \
  --name web-app \
  -p 8080:80 \
  docker.io/library/nginx:latest

# Verify the container is running
podman ps

# Test from the instance
curl http://localhost:8080
```

Remember to configure your EC2 security group to allow traffic on port 8080:

```bash
# List the container ports
podman port web-app

# Clean up
podman stop web-app
podman rm web-app
```

## Using Podman with AWS ECR

Pull images from Amazon Elastic Container Registry:

```bash
# Install the AWS CLI if not already present
sudo dnf install -y aws-cli   # AL2023

# Authenticate with ECR (replace region and account ID)
aws ecr get-login-password --region us-east-1 | \
  podman login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Pull an image from ECR
podman pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

## Auto-Starting Containers with systemd

Create a systemd service for containers that should survive reboots:

```bash
# Generate a systemd unit for a running container
podman generate systemd --new --name web-app > ~/.config/systemd/user/container-web-app.service

# Reload and enable the service
systemctl --user daemon-reload
systemctl --user enable container-web-app.service

# Enable lingering so user services start at boot
sudo loginctl enable-linger ec2-user
```

## Troubleshooting

If Podman fails with namespace errors on Amazon Linux 2:

```bash
# Check user namespace support
sysctl user.max_user_namespaces

# If the value is 0, set it
sudo sysctl -w user.max_user_namespaces=28633
```

If you cannot pull images from Docker Hub:

```bash
# Configure registries
sudo tee /etc/containers/registries.conf.d/docker.conf <<EOF
[registries.search]
registries = ['docker.io']
EOF
```

## Summary

Podman is well-supported on Amazon Linux, especially on AL2023 where it is the recommended container runtime. Integration with AWS ECR and systemd makes it suitable for production workloads on EC2. For Amazon Linux 2, the extras library provides access to Podman with minimal extra configuration.
