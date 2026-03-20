# How to Set Up Elemental with iPXE Boot

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, IPXE, Network Boot, Edge, Kubernetes

Description: Configure iPXE network boot for Elemental nodes to enable zero-touch provisioning without physical media.

## Introduction

iPXE network boot enables you to provision Elemental nodes without creating physical boot media for each machine. Machines boot from the network, load the Elemental environment, register with Rancher, and install the OS automatically. This is ideal for data center and factory environments where network infrastructure is available.

## Prerequisites

- DHCP server (for PXE option configuration)
- TFTP or HTTP server for serving boot files
- Elemental OS image available
- Network with PXE boot support

## Step 1: Set Up the Boot Server

```bash
# Install required services on your boot server

zypper install dhcp-server tftpd

# Or on Ubuntu/Debian
apt-get install isc-dhcp-server tftpd-hpa

# Create TFTP directory
mkdir -p /srv/tftp/elemental
```

## Step 2: Configure DHCP for PXE

```bash
# /etc/dhcp/dhcpd.conf
cat > /etc/dhcp/dhcpd.conf << 'EOF'
option domain-name "example.com";
option domain-name-servers 8.8.8.8, 8.8.4.4;

subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;

  # iPXE boot configuration
  if exists user-class and option user-class = "iPXE" {
    # iPXE already loaded, serve the boot script
    filename "http://192.168.1.10:8080/boot.ipxe";
  } else {
    # First boot, serve iPXE loader
    filename "ipxe.efi";
    next-server 192.168.1.10;
  }
}
EOF

systemctl restart dhcpd
```

## Step 3: Create iPXE Boot Script

```bash
# Create the iPXE boot script
cat > /srv/http/boot.ipxe << 'EOF'
#!ipxe

# Set variables
set base-url http://192.168.1.10:8080/elemental
set registration-url https://rancher.example.com/v1/elemental/registration/your-token

echo Booting Elemental OS...

# Download and boot kernel
kernel ${base-url}/vmlinuz \
  rd.neednet=1 \
  ip=dhcp \
  root=live:${base-url}/rootfs.squashfs \
  console=tty1 \
  console=ttyS0,115200n8 \
  elemental.registration.url=${registration-url} \
  rd.debug

# Load initrd
initrd ${base-url}/initrd

# Boot
boot
EOF
```

## Step 4: Serve Elemental Boot Files via HTTP

```bash
# Extract boot files from Elemental ISO
mkdir -p /srv/http/elemental
mount -o loop elemental-seed.iso /mnt/iso

# Copy boot files
cp /mnt/iso/boot/vmlinuz /srv/http/elemental/
cp /mnt/iso/boot/initrd /srv/http/elemental/
cp /mnt/iso/rootfs.squashfs /srv/http/elemental/

umount /mnt/iso

# Start HTTP server
python3 -m http.server 8080 --directory /srv/http
```

## Step 5: Configure iPXE with Registration Parameters

```bash
# Advanced iPXE script with Elemental registration
cat > /srv/http/boot-elemental.ipxe << 'EOF'
#!ipxe

echo === Elemental Auto-Registration Boot ===

# Network configuration
dhcp

# Base URL for boot files
set base http://192.168.1.10:8080

# Elemental registration parameters
set reg-url https://rancher.example.com/v1/elemental/registration/TOKEN
set ca-cert-url http://192.168.1.10:8080/ca.pem

# Boot the Elemental OS
kernel ${base}/vmlinuz \
  initrd=initrd \
  root=live:${base}/rootfs.squashfs \
  rd.live.overlay.overlayfs=1 \
  console=tty1 \
  console=ttyS0,115200n8 \
  elemental.registration.uri=${reg-url} \
  elemental.registration.ca-cert-url=${ca-cert-url} \
  elemental.install.device=/dev/sda \
  elemental.install.reboot=true

initrd ${base}/initrd
boot
EOF
```

## Step 6: Test the Setup

```bash
# Test iPXE boot in a VM
qemu-system-x86_64 \
  -m 4096 \
  -netdev user,id=net0,net=192.168.1.0/24,dhcpstart=192.168.1.100 \
  -device virtio-net-pci,netdev=net0 \
  -boot n \
  -nographic
```

## Monitoring iPXE Registrations

```bash
# Watch for new machine registrations
kubectl get machineinventory -n fleet-default --watch

# Check DHCP leases
cat /var/lib/dhcpd/dhcpd.leases | grep "binding state active"
```

## Conclusion

iPXE network boot with Elemental eliminates the need for physical boot media, enabling fully automated provisioning at scale. By embedding registration parameters directly in the iPXE boot script, machines can go from bare metal to a registered Kubernetes node without any manual intervention, making it ideal for large-scale deployments.
