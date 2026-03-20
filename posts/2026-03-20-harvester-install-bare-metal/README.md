# How to Install Harvester on Bare Metal

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Bare Metal

Description: A step-by-step guide to installing Harvester HCI on bare metal servers for production hyperconverged infrastructure.

## Introduction

Harvester is an open-source hyperconverged infrastructure (HCI) solution built on Kubernetes. It combines compute, storage, and networking into a unified platform, making it ideal for running virtual machines alongside containerized workloads. Installing Harvester on bare metal gives you full hardware control and the best performance characteristics.

This guide walks you through the complete bare metal installation process, from pre-installation checks to a running Harvester cluster.

## Prerequisites

Before you begin, ensure you meet these hardware and software requirements:

**Minimum Hardware Requirements (per node):**
- CPU: 8 cores, x86_64 architecture with hardware virtualization support (Intel VT-x or AMD-V)
- RAM: 32 GB
- Storage: 250 GB SSD for the OS disk, plus additional disks for VM storage
- Network: Two NICs recommended (one for management, one for VM traffic)

**Software Requirements:**
- Harvester ISO (download from the [official releases page](https://github.com/harvester/harvester/releases))
- A USB drive (8 GB or larger) or a PXE boot environment

**Network Requirements:**
- A static IP address or DHCP reservation for the management network
- DNS resolution for the cluster VIP (Virtual IP)
- NTP access for time synchronization

## Step 1: Download the Harvester ISO

Download the latest stable Harvester ISO from the GitHub releases page:

```bash
# Download the latest Harvester ISO
wget https://releases.rancher.com/harvester/v1.3.0/harvester-v1.3.0-amd64.iso

# Verify the checksum
wget https://releases.rancher.com/harvester/v1.3.0/harvester-v1.3.0-amd64.iso.sha512
sha512sum -c harvester-v1.3.0-amd64.iso.sha512
```

## Step 2: Create Bootable Media

Use `dd` on Linux or Rufus on Windows to write the ISO to a USB drive:

```bash
# On Linux: Identify your USB device (replace /dev/sdX with your device)
lsblk

# Write the ISO to the USB drive (this will erase all data on the USB)
sudo dd if=harvester-v1.3.0-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

## Step 3: Configure BIOS/UEFI Settings

Before booting from the installation media, configure your server's BIOS/UEFI:

1. Enable hardware virtualization (Intel VT-x / AMD-V)
2. Enable IOMMU if you plan to use PCI passthrough
3. Set the boot order to boot from USB first
4. Disable Secure Boot (Harvester may not be compatible with all Secure Boot configurations)
5. Enable Wake-on-LAN if needed for remote management

## Step 4: Boot from Installation Media

1. Insert the USB drive into the server
2. Power on the server and enter the boot menu (typically F12, F10, or Del)
3. Select the USB drive as the boot device
4. The Harvester installer GRUB menu will appear — select **Install Harvester**

## Step 5: Run the Interactive Installer

The Harvester installer is a text-based UI that guides you through the setup:

### Installation Mode
Choose **Create a new Harvester cluster** for the first node, or **Join an existing Harvester cluster** for additional nodes.

### Network Configuration
```
# Example network settings for the management interface
Management NIC:   eth0
IP Address:       192.168.1.10/24
Gateway:          192.168.1.1
DNS:              8.8.8.8, 8.8.4.4
```

### Cluster VIP
The Virtual IP (VIP) is the highly available endpoint for the cluster API and UI:

```
Cluster VIP: 192.168.1.100
```

This VIP must be on the same subnet as the management network and must not be assigned to any other device.

### Storage Configuration
Select the disk(s) for the Harvester OS installation. Harvester will use the remaining disks for VM storage via Longhorn.

```
OS Disk:      /dev/sda  (250 GB SSD - for the operating system)
Data Disks:   /dev/sdb, /dev/sdc  (auto-detected by Longhorn)
```

### Set Passwords
Configure the cluster admin password and the `rancher` user password for node SSH access.

## Step 6: Complete Installation

The installer will:
1. Partition and format the selected OS disk
2. Install the Harvester OS (based on openSUSE Leap Micro)
3. Configure Kubernetes (RKE2) and all Harvester components
4. Reboot the node

Installation typically takes 10–20 minutes depending on hardware speed.

## Step 7: Access the Harvester UI

Once the node reboots, you can access the Harvester dashboard:

1. Open a browser and navigate to `https://<CLUSTER_VIP>`
2. Accept the self-signed certificate warning
3. Log in with `admin` and the password you set during installation

```bash
# Alternatively, access via kubectl
# The kubeconfig is available on the node at:
cat /etc/rancher/rke2/rke2.yaml

# Set KUBECONFIG and verify cluster health
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
kubectl get nodes
kubectl get pods -A
```

## Step 8: Verify the Installation

After logging in, verify your cluster is healthy:

```bash
# Check all nodes are Ready
kubectl get nodes -o wide

# Check Harvester system pods are running
kubectl get pods -n harvester-system

# Check Longhorn storage is healthy
kubectl get pods -n longhorn-system
```

In the UI, navigate to **Dashboard** and confirm:
- Node status shows **Ready**
- Storage shows available capacity
- No critical alerts are present

## Post-Installation Configuration

After the base installation, consider these next steps:

- **Add additional nodes** to increase capacity and redundancy
- **Configure backup targets** (NFS or S3) for VM backups
- **Set up VLAN networks** for VM network isolation
- **Install Rancher** on top of Harvester for advanced cluster management
- **Configure monitoring** with the built-in Grafana/Prometheus stack

## Conclusion

You now have a running single-node Harvester cluster on bare metal. While a single node is useful for development and testing, production deployments should use at least three nodes for high availability. The bare metal installation gives you full access to hardware capabilities including SR-IOV, PCI passthrough, and NUMA-aware scheduling. From here, you can start creating VM images, defining networks, and deploying virtual machines through the intuitive Harvester UI or Kubernetes-native APIs.
