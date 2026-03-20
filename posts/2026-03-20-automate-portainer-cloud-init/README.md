# How to Automate Portainer Deployment with Cloud-Init

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Cloud-init, Automation, Cloud, Infrastructure as Code

Description: Use cloud-init to automatically install Docker and deploy Portainer on new cloud instances or VMs with zero manual intervention.

## Introduction

Cloud-init is the industry-standard multi-distribution method for cross-platform cloud instance initialization. When a new virtual machine or cloud instance boots for the first time, cloud-init runs your configuration scripts to set up the system. By combining cloud-init with Portainer, you can have a fully configured container management platform ready within minutes of launching a new instance - completely automated.

## Prerequisites

- Cloud provider account (AWS, GCP, Azure, DigitalOcean, Hetzner, etc.)
- Basic understanding of YAML
- SSH key pair for instance access
- Ubuntu 22.04 or Debian 12 base image

## Step 1: Basic Cloud-Init Structure

Cloud-init uses a YAML file called user-data that's passed to instances at launch:

```yaml
#cloud-config
# This comment is required - it identifies this as a cloud-init file

# System configuration

package_update: true
package_upgrade: true
packages:
  - curl
  - wget
  - git
  - htop
  - unzip

# Create users
users:
  - default
  - name: portainer-admin
    groups: [docker, sudo]
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh_authorized_keys:
      - "ssh-rsa AAAA... your-public-key"
```

## Step 2: Complete Portainer Deployment Cloud-Init

This cloud-init configuration installs Docker and deploys Portainer in one step:

```yaml
#cloud-config
# Portainer Auto-Deployment Cloud-Init Configuration
# Compatible with Ubuntu 22.04 and Debian 12

package_update: true
package_upgrade: false

packages:
  - curl
  - wget
  - apt-transport-https
  - ca-certificates
  - gnupg
  - lsb-release

# Write configuration files
write_files:
  # Docker daemon configuration
  - path: /etc/docker/daemon.json
    content: |
      {
        "log-driver": "json-file",
        "log-opts": {
          "max-size": "100m",
          "max-file": "3"
        },
        "storage-driver": "overlay2",
        "live-restore": true
      }
    owner: root:root
    permissions: '0644'
  
  # Portainer docker-compose configuration
  - path: /opt/portainer/docker-compose.yml
    content: |
      version: "3.8"
      services:
        portainer:
          image: portainer/portainer-ce:2.19.4
          container_name: portainer
          restart: always
          ports:
            - "9000:9000"
            - "9443:9443"
            - "8000:8000"
          volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - portainer-data:/data
          environment:
            - PORTAINER_ADMIN_PASSWORD_HASH=${PORTAINER_ADMIN_HASH}
          logging:
            driver: json-file
            options:
              max-size: "50m"
              max-file: "3"
      volumes:
        portainer-data:
    owner: root:root
    permissions: '0644'
  
  # Portainer initialization script
  - path: /opt/portainer/init.sh
    content: |
      #!/bin/bash
      set -euo pipefail
      
      echo "=== Installing Docker ==="
      curl -fsSL https://get.docker.com | sh
      systemctl enable docker
      systemctl start docker
      
      echo "=== Installing Docker Compose Plugin ==="
      apt-get install -y docker-compose-plugin 2>/dev/null || true
      
      echo "=== Starting Portainer ==="
      cd /opt/portainer
      docker compose up -d
      
      echo "=== Waiting for Portainer to start ==="
      for i in $(seq 1 30); do
        if curl -sk https://localhost:9443/api/system/status | grep -q "Version"; then
          echo "Portainer is ready!"
          break
        fi
        echo "Waiting... ($i/30)"
        sleep 5
      done
      
      echo "=== Portainer deployment complete ==="
      echo "Access Portainer at: https://$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4):9443"
    owner: root:root
    permissions: '0755'
  
  # Firewall configuration
  - path: /opt/portainer/setup-firewall.sh
    content: |
      #!/bin/bash
      # Configure UFW firewall
      ufw allow ssh
      ufw allow 9443/tcp comment "Portainer HTTPS"
      ufw allow 9000/tcp comment "Portainer HTTP"
      ufw allow 8000/tcp comment "Portainer Edge Agent"
      ufw --force enable
    owner: root:root
    permissions: '0755'

# Run commands in order
runcmd:
  # Install Docker and deploy Portainer
  - [bash, /opt/portainer/init.sh]
  
  # Configure firewall
  - [bash, /opt/portainer/setup-firewall.sh]
  
  # Add default user to docker group
  - [usermod, -aG, docker, ubuntu]
  
  # Set up log rotation
  - |
    cat > /etc/logrotate.d/portainer << 'EOF'
    /opt/portainer/logs/*.log {
        daily
        rotate 7
        compress
        delaycompress
        missingok
        notifempty
    }
    EOF

# Send notification when complete
phone_home:
  url: "https://your-webhook.example.com/cloud-init/complete"
  tries: 3
```

## Step 3: AWS EC2 Deployment

Launch an EC2 instance with the cloud-init configuration:

```bash
# Save cloud-init config
cat > portainer-userdata.yaml << 'USERDATA'
#cloud-config
# ... (paste the config from Step 2)
USERDATA

# Launch EC2 instance with user-data
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name my-key-pair \
  --security-group-ids sg-12345678 \
  --user-data file://portainer-userdata.yaml \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=portainer-server}]' \
  --region us-east-1

# Check instance status
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=portainer-server" \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,IP:PublicIpAddress}'
```

## Step 4: Hetzner Cloud Deployment

```bash
# Using Hetzner Cloud CLI (hcloud)
hcloud server create \
  --name portainer-server \
  --type cx21 \
  --image ubuntu-22.04 \
  --ssh-key my-key \
  --user-data-from-file portainer-userdata.yaml \
  --location nbg1

# Watch the server creation
hcloud server create --help
```

## Step 5: DigitalOcean Droplet Deployment

```bash
# Using DigitalOcean CLI (doctl)
doctl compute droplet create portainer-server \
  --region nyc3 \
  --size s-2vcpu-2gb \
  --image ubuntu-22-04-x64 \
  --ssh-keys your-key-id \
  --user-data-file portainer-userdata.yaml \
  --wait
```

## Step 6: Cloud-Init for Portainer Edge Agent

Deploy the lightweight Edge Agent on edge nodes:

```yaml
#cloud-config
# Portainer Edge Agent Cloud-Init
# Use this for remote edge nodes that connect back to central Portainer

packages:
  - curl

write_files:
  - path: /opt/portainer-agent/deploy.sh
    content: |
      #!/bin/bash
      # Install Docker
      curl -fsSL https://get.docker.com | sh
      
      # Deploy Portainer Edge Agent
      # Replace EDGE_KEY with your actual edge key from Portainer UI
      docker run -d \
        --name portainer-agent \
        --restart always \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v /var/lib/docker/volumes:/var/lib/docker/volumes \
        -p 9001:9001 \
        portainer/agent:latest \
        -H tcp://portainer.example.com:8000 \
        --key ${PORTAINER_EDGE_KEY}
    permissions: '0755'

runcmd:
  - [bash, /opt/portainer-agent/deploy.sh]
```

## Verifying Cloud-Init Execution

```bash
# Check cloud-init status
cloud-init status

# View cloud-init logs
cat /var/log/cloud-init.log
cat /var/log/cloud-init-output.log

# Check if Portainer is running
docker ps | grep portainer
curl -sk https://localhost:9443/api/system/status
```

## Conclusion

Cloud-init automates Portainer deployment to a point where new instances are fully configured with no manual intervention. Whether you're launching EC2 instances for development environments, provisioning edge nodes in remote locations, or building immutable infrastructure, cloud-init ensures consistent, reproducible Portainer deployments. Combined with cloud provider APIs or infrastructure as code tools like Terraform, you can scale this approach to provision Portainer across entire fleets of servers automatically.
