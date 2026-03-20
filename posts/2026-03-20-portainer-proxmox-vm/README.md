# How to Deploy Portainer on a Proxmox Virtual Machine

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Proxmox, Virtual Machine, Docker, Self-Hosted, Home Lab

Description: Create a dedicated Ubuntu VM in Proxmox and deploy Portainer with proper resource allocation and ZFS or LVM storage for reliable container management.

## Introduction

Proxmox VE is a popular open-source virtualization platform for home labs and small data centers. Deploying Portainer in a dedicated Proxmox VM gives you clean separation from the host OS, easy snapshot and backup capabilities, and the full Docker experience. This guide covers creating the VM and deploying Portainer.

## Prerequisites

- Proxmox VE 8.x installed
- Ubuntu 24.04 LTS ISO uploaded to Proxmox
- At least 4GB RAM and 2 vCPUs available to allocate
- Network bridge configured in Proxmox

## Step 1: Upload Ubuntu ISO

1. In Proxmox WebUI, navigate to your node > **local (storage)**
2. Click **ISO Images > Upload**
3. Upload Ubuntu 24.04 LTS server ISO
4. Or download directly:

```bash
# On Proxmox host via SSH

wget -P /var/lib/vz/template/iso/ \
  https://releases.ubuntu.com/24.04/ubuntu-24.04.1-live-server-amd64.iso
```

## Step 2: Create the VM

### Via Proxmox WebUI

1. Click **Create VM**
2. **General**: Node: your node, VM ID: 200, Name: `portainer-vm`
3. **OS**: Select uploaded Ubuntu ISO, Type: Linux, Version: 6.x
4. **System**: Leave defaults (OVMF BIOS optional, SeaBIOS works fine)
5. **Disks**: Add disk, Storage: local-lvm (or your preferred storage), Size: 32GB
6. **CPU**: Sockets: 1, Cores: 2
7. **Memory**: 4096 MB (4GB)
8. **Network**: Bridge: vmbr0, Model: VirtIO
9. Click **Finish**

### Via Proxmox CLI (pvesh)

```bash
# SSH into Proxmox host
ssh root@<proxmox-ip>

# Create VM
qm create 200 \
  --name portainer-vm \
  --memory 4096 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --ide2 local:iso/ubuntu-24.04-live-server-amd64.iso,media=cdrom \
  --scsi0 local-lvm:32 \
  --scsihw virtio-scsi-pci \
  --boot c --bootdisk scsi0 \
  --agent enabled=1 \
  --ostype l26

# Start the VM
qm start 200
```

## Step 3: Install Ubuntu in the VM

Connect via Proxmox console (VNC) or:

```bash
# Open console via Proxmox WebUI
# Navigate to VM > Console
```

Install Ubuntu with:
- Minimal installation
- OpenSSH server: Yes
- No LVM/RAID needed inside VM (Proxmox handles this)

After installation:

```bash
# SSH to VM
ssh ubuntu@<vm-ip>

# Update
sudo apt update && sudo apt upgrade -y

# Install QEMU guest agent for better Proxmox integration
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

## Step 4: Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
newgrp docker
sudo systemctl enable docker
```

## Step 5: Deploy Portainer

```bash
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

## Step 6: Configure Proxmox Snapshots

Back up your VM with Proxmox's built-in snapshot feature:

```bash
# Take a snapshot via Proxmox CLI
qm snapshot 200 pre-portainer-install --description "Before Portainer installation"

# List snapshots
qm listsnapshot 200
```

Or configure automated backups:

1. In Proxmox, navigate to **Datacenter > Backup**
2. Click **Add**
3. Select your VM (ID 200)
4. Schedule: Daily at 2:00 AM
5. Storage: select your backup storage
6. Mode: Snapshot

## Step 7: Add Proxmox Disk for Docker Data

```bash
# On Proxmox host, add a disk to the VM
qm set 200 --scsi1 local-lvm:50

# In the VM, format and mount
sudo fdisk -l  # Find /dev/sdb
sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /data

# Get UUID for fstab
sudo blkid /dev/sdb

echo 'UUID=<your-uuid> /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
sudo mount -a

# Reconfigure Docker
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "data-root": "/data/docker"
}
EOF
sudo systemctl stop docker
sudo rsync -aP /var/lib/docker/ /data/docker/
sudo systemctl start docker
```

## Conclusion

Running Portainer in a Proxmox VM combines the best of both worlds: Proxmox's enterprise-grade snapshot, backup, and migration capabilities with Docker's container flexibility. The VM approach provides clean isolation from the Proxmox host OS and makes it easy to clone the environment or migrate it to another Proxmox node. Qemu guest agent integration ensures Proxmox has accurate VM state information.
