# How to Deploy Portainer on Hetzner Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Hetzner, Docker, Cloud, VPS

Description: Learn how to deploy Portainer on Hetzner Cloud servers, one of the most cost-effective cloud providers, with firewall rules, cloud-init setup, and volume attachment.

## Why Hetzner for Portainer?

Hetzner Cloud offers some of the best price-to-performance ratios in cloud hosting. A CX22 server (2 vCPU, 4GB RAM) costs ~€4/month — comparable performance to a $24/month Droplet elsewhere.

## Step 1: Create a Server via hcloud CLI

```bash
# Install hcloud CLI
brew install hcloud    # macOS
# or download from https://github.com/hetznercloud/cli

# Authenticate
hcloud context create portainer

# Create server
hcloud server create \
  --name portainer-server \
  --type cx22 \
  --image ubuntu-22.04 \
  --location nbg1 \
  --ssh-key my-ssh-key \
  --user-data-from-file /tmp/cloud-init.yaml

# Get IP
hcloud server ip portainer-server
```

Cloud-init file (`/tmp/cloud-init.yaml`):

```yaml
#cloud-config
runcmd:
  - curl -fsSL https://get.docker.com | sh
  - docker volume create portainer_data
  - docker run -d -p 8000:8000 -p 9443:9443
    --name portainer --restart=always
    -v /var/run/docker.sock:/var/run/docker.sock
    -v portainer_data:/data
    portainer/portainer-ce:latest
```

## Step 2: Create a Firewall

```bash
# Create firewall
hcloud firewall create --name portainer-fw

# Add inbound rules
hcloud firewall add-rule portainer-fw \
  --direction in --protocol tcp --port 22 --source-ips 0.0.0.0/0

hcloud firewall add-rule portainer-fw \
  --direction in --protocol tcp --port 9443 --source-ips 0.0.0.0/0

hcloud firewall add-rule portainer-fw \
  --direction in --protocol tcp --port 80 --source-ips 0.0.0.0/0

hcloud firewall add-rule portainer-fw \
  --direction in --protocol tcp --port 443 --source-ips 0.0.0.0/0

# Apply to server
hcloud firewall apply-to-resource portainer-fw \
  --type server --server portainer-server
```

## Step 3: Attach a Volume for Data Persistence

```bash
# Create a volume (separate from server's root disk)
hcloud volume create \
  --name portainer-data-vol \
  --size 20 \
  --server portainer-server \
  --automount \
  --format ext4

# The volume will be mounted at /mnt/HC_Volume_<ID>
# SSH in and check:
df -h | grep HC_Volume
```

Use the volume for Docker data:

```bash
# SSH into server
ssh root@<SERVER_IP>

# Move Docker data root to the volume
cat > /etc/docker/daemon.json << 'EOF'
{
  "data-root": "/mnt/HC_Volume_123456789"
}
EOF

systemctl restart docker
```

## Step 4: Configure Hetzner DNS

```bash
# Using Hetzner DNS API
ZONE_ID=$(curl -s "https://dns.hetzner.com/api/v1/zones" \
  -H "Auth-API-Token: $HETZNER_DNS_TOKEN" | \
  jq -r '.zones[] | select(.name == "yourdomain.com") | .id')

# Create A record
curl -X POST "https://dns.hetzner.com/api/v1/records" \
  -H "Auth-API-Token: $HETZNER_DNS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"value\": \"$(hcloud server ip portainer-server)\", \"ttl\": 300, \"type\": \"A\", \"name\": \"portainer\", \"zone_id\": \"$ZONE_ID\"}"
```

## MTU Configuration for Docker Swarm on Hetzner

Hetzner Cloud uses MTU 1450 for its network. Docker overlay networks default to MTU 1500 and can cause packet fragmentation:

```json
# /etc/docker/daemon.json
{
  "mtu": 1450
}
```

See also: fix-swarm-mtu-issues-portainer-hetzner guide.

## Recommended Server Types

| Type | vCPU | RAM | Price | Use Case |
|------|------|-----|-------|----------|
| CX22 | 2 | 4GB | ~€4/mo | Personal |
| CX32 | 4 | 8GB | ~€7/mo | Team |
| CX42 | 8 | 16GB | ~€14/mo | Production |

## Conclusion

Hetzner Cloud delivers exceptional value for Portainer deployments. The combination of cloud-init automated setup, hcloud CLI management, and Hetzner's firewall and volume features makes it easy to build a production-ready Portainer environment at a fraction of the cost of AWS or Azure.
