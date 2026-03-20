# How to Deploy Portainer in a Proxmox LXC Container

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Proxmox, LXC, Docker, Self-Hosted, Home Lab

Description: Run Portainer inside a Proxmox LXC container for a lightweight alternative to full VMs, using less RAM while maintaining Docker functionality.

## Introduction

Proxmox LXC containers use significantly less RAM than full VMs (typically 50-100MB less overhead per container). Running Docker and Portainer inside a privileged LXC container is a popular home lab approach for running many services on a single Proxmox host. This guide covers creating an LXC container with Docker support.

## Prerequisites

- Proxmox VE 8.x
- Ubuntu 22.04 or Debian 12 LXC template downloaded
- At least 512MB RAM available (2GB recommended for Portainer + containers)

## Step 1: Download LXC Template

```bash
# SSH to Proxmox host
ssh root@<proxmox-ip>

# Download Ubuntu 22.04 LXC template
pveam download local ubuntu-22.04-standard_22.04-1_amd64.tar.zst

# Or update available templates and choose
pveam update
pveam available | grep ubuntu
```

## Step 2: Create the LXC Container

### Via Proxmox WebUI

1. Click **Create CT**
2. **General**: CT ID: 300, Hostname: `portainer-lxc`, Password: set strong password
3. **Template**: Select ubuntu-22.04 template
4. **Root Disk**: 20GB on local-lvm
5. **CPU**: 2 cores
6. **Memory**: 2048 MB RAM, 1024 MB Swap
7. **Network**: Name: eth0, Bridge: vmbr0, IPv4: DHCP or static
8. **DNS**: Leave default
9. **Confirm**: Check **Start after created**

### Via Proxmox CLI

```bash
pct create 300 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname portainer-lxc \
  --memory 2048 \
  --swap 1024 \
  --cores 2 \
  --rootfs local-lvm:20 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 0 \
  --features nesting=1,keyctl=1 \
  --start 1
```

**Important**: `--unprivileged 0` makes it privileged (required for Docker), and `--features nesting=1` enables Docker nesting.

## Step 3: Configure LXC for Docker

```bash
# Stop the container
pct stop 300

# Edit LXC config to add required settings
nano /etc/pve/lxc/300.conf

# Add these lines:
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
lxc.mount.auto: proc:rw sys:rw
```

Or add via command:

```bash
cat >> /etc/pve/lxc/300.conf << 'EOF'
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
EOF

# Start container
pct start 300
```

## Step 4: Install Docker in LXC

```bash
# Enter the container
pct enter 300

# Update
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sh
systemctl enable docker

# Verify Docker works
docker run hello-world
```

## Step 5: Fix Common Docker-in-LXC Issues

### overlay2 Storage Driver

If overlay2 fails, use a different storage driver:

```bash
tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
systemctl restart docker
```

If overlay2 still fails (older kernels), try `fuse-overlayfs`:

```bash
apt install -y fuse-overlayfs

tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "storage-driver": "fuse-overlayfs"
}
EOF
systemctl restart docker
```

## Step 6: Deploy Portainer

```bash
# Inside the LXC container
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 7: Proxmox Backup for LXC

```bash
# Backup the LXC container (on Proxmox host)
vzdump 300 --compress zstd --storage local --mode snapshot

# Scheduled backup via Proxmox WebUI:
# Datacenter > Backup > Add
# Select CT 300, schedule daily
```

## Resource Comparison: LXC vs VM

| Feature | LXC Container | Full VM |
|---------|--------------|---------|
| RAM overhead | ~50MB | ~300MB |
| Boot time | 2-3 seconds | 20-30 seconds |
| Snapshot support | Yes | Yes |
| Live migration | Limited | Full support |
| Docker compatibility | Good (privileged) | Full |
| Security isolation | Lower | Higher |

## Conclusion

Proxmox LXC containers with Docker and Portainer offer an excellent balance of resource efficiency and functionality for home labs. The privileged LXC approach works well for trusted internal workloads, and the lower resource overhead means you can run more services on the same Proxmox host. Combined with Proxmox's backup and snapshot features, this is a robust home lab setup.
