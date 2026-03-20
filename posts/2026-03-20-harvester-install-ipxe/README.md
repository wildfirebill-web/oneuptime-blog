# How to Install Harvester with iPXE

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, IPXE, PXE Boot

Description: Learn how to automate Harvester installation across multiple bare metal servers using iPXE network booting for scalable data center deployments.

## Introduction

iPXE network booting allows you to install Harvester on multiple servers without physically inserting USB drives into each machine. This approach is ideal for data centers where you need to provision tens or hundreds of nodes consistently and efficiently. iPXE extends the traditional PXE (Preboot Execution Environment) standard with support for HTTP, scripting, and more advanced network protocols.

## Prerequisites

- A DHCP server on your network
- An HTTP/TFTP server to host the boot files
- Servers with network boot (PXE) enabled in BIOS/UEFI
- Harvester ISO and kernel files downloaded
- Basic knowledge of network configuration

## Architecture Overview

The iPXE boot chain works as follows:

```mermaid
graph LR
    A[Server Powers On] --> B[Gets IP from DHCP]
    B --> C[Downloads iPXE bootloader via TFTP]
    C --> D[Runs iPXE script from HTTP]
    D --> E[Downloads Harvester kernel + initrd]
    E --> F[Boots into Harvester Installer]
```

## Step 1: Set Up the HTTP File Server

Create a directory structure to serve the Harvester boot files:

```bash
# Create the directory structure

sudo mkdir -p /var/www/html/harvester/v1.3.0

# Download the required boot artifacts
cd /var/www/html/harvester/v1.3.0

# Download the ISO
wget https://releases.rancher.com/harvester/v1.3.0/harvester-v1.3.0-amd64.iso

# Download the kernel and initrd for netboot
wget https://releases.rancher.com/harvester/v1.3.0/harvester-v1.3.0-vmlinuz-amd64
wget https://releases.rancher.com/harvester/v1.3.0/harvester-v1.3.0-initrd-amd64
wget https://releases.rancher.com/harvester/v1.3.0/harvester-v1.3.0-rootfs-amd64.squashfs
```

## Step 2: Set Up TFTP Server

Install and configure a TFTP server to serve the iPXE bootloader:

```bash
# Install TFTP server (Ubuntu/Debian)
sudo apt-get install -y tftpd-hpa

# Install iPXE binaries
sudo apt-get install -y ipxe

# Copy iPXE binaries to the TFTP root
sudo cp /usr/lib/ipxe/undionly.kpxe /var/lib/tftpboot/
sudo cp /usr/lib/ipxe/ipxe.efi /var/lib/tftpboot/

# Start and enable the TFTP service
sudo systemctl enable --now tftpd-hpa
```

## Step 3: Configure DHCP Server

Configure your DHCP server to point clients to the iPXE bootloader. Below is an example using `dnsmasq`:

```ini
# /etc/dnsmasq.conf - DHCP and TFTP configuration

# DHCP range
dhcp-range=192.168.1.100,192.168.1.200,12h

# TFTP server (same host in this example)
dhcp-boot=tag:!ipxe,undionly.kpxe

# For UEFI clients, serve the EFI bootloader
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-boot=tag:efi-x86_64,ipxe.efi

# Once iPXE is loaded, redirect to the boot script
dhcp-boot=tag:ipxe,http://192.168.1.50/harvester/boot.ipxe

# Enable TFTP
enable-tftp
tftp-root=/var/lib/tftpboot
```

## Step 4: Create the iPXE Boot Script

Create the main iPXE script that Harvester nodes will load:

```bash
# /var/www/html/harvester/boot.ipxe

#!ipxe

# Set the Harvester version and server URL
set harvester_version v1.3.0
set base_url http://192.168.1.50/harvester/${harvester_version}

echo Booting Harvester ${harvester_version}...

# Kernel and initrd paths
kernel ${base_url}/harvester-${harvester_version}-vmlinuz-amd64 \
    ip=dhcp \
    rd.cos.disable \
    root=live:${base_url}/harvester-${harvester_version}-rootfs-amd64.squashfs \
    harvester.install.automatic=true \
    harvester.install.config_url=http://192.168.1.50/harvester/config.yaml \
    console=ttyS0,115200n8 \
    console=tty0 \
    net.ifnames=1

initrd ${base_url}/harvester-${harvester_version}-initrd-amd64

boot
```

## Step 5: Create the Harvester Configuration File

For unattended (automated) installation, create a configuration YAML:

```yaml
# /var/www/html/harvester/config.yaml
# Harvester automated installation configuration

scheme_version: 1

# Installation settings
install:
  # The device to install Harvester OS onto
  device: /dev/sda
  # Automatically confirm installation (no interactive prompts)
  automatic: true

os:
  # Hostname for this node
  hostname: harvester-node-01
  # SSH public key for the rancher user
  ssh_authorized_keys:
    - ssh-rsa AAAA... your-public-key-here
  # Password for the rancher user (hashed)
  password: "$6$rounds=4096$salt$hash"
  # NTP servers
  ntp_servers:
    - pool.ntp.org
  # DNS servers
  dns_nameservers:
    - 8.8.8.8
    - 8.8.4.4

# Network configuration
network:
  interfaces:
    - name: eth0
      hwAddr: ""
      mtu: 1500
  bonds:
    - name: harvester-mgmt
      mode: active-backup
      slaves:
        - eth0
  vlans: []

# Harvester cluster settings
harvester:
  # Mode: create (first node) or join (subsequent nodes)
  mode: create
  # Management interface for the cluster
  management_interface:
    interfaces:
      - name: eth0
    method: dhcp
  # Cluster token (shared secret for joining nodes)
  token: "your-secret-cluster-token"
  # Virtual IP for the cluster (used for API and UI access)
  vip: 192.168.1.100
  vip_mode: static
  # Admin password for the Harvester UI
  password: "YourAdminPassword"
```

## Step 6: Test the iPXE Boot

Power on a server that is configured to network boot:

```bash
# Monitor the DHCP leases to verify the server got an IP
tail -f /var/log/syslog | grep dnsmasq

# Watch the TFTP server logs to see the boot file transfers
tail -f /var/log/syslog | grep tftpd

# Monitor the HTTP server access log for kernel/initrd downloads
tail -f /var/log/apache2/access.log
```

## Automating Multi-Node Deployments

For deploying multiple nodes, use different config files per node or use templates with hostname derivation:

```bash
# /var/www/html/harvester/config-join.yaml
# Configuration for additional nodes joining the cluster

scheme_version: 1

install:
  device: /dev/sda
  automatic: true

harvester:
  mode: join
  # URL of the first (seed) node
  server_url: https://192.168.1.100:443
  token: "your-secret-cluster-token"
  management_interface:
    interfaces:
      - name: eth0
    method: dhcp
```

## Troubleshooting

**Node can't reach the HTTP server:**
```bash
# Check that the HTTP server is listening
ss -tlnp | grep ':80'

# Test from another machine
curl -I http://192.168.1.50/harvester/boot.ipxe
```

**DHCP not assigning the correct boot file:**
```bash
# Capture DHCP traffic to diagnose
tcpdump -i eth0 port 67 or port 68 -n
```

## Conclusion

iPXE network booting transforms Harvester deployment from a manual, one-server-at-a-time process into a scalable, automated workflow. By combining iPXE with a configuration YAML, you can provision an entire Harvester cluster without touching a single server physically. This approach integrates naturally with infrastructure-as-code practices and can be extended to support zero-touch provisioning with tools like Tinkerbell or Terraform.
