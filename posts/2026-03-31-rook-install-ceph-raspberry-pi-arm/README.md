# How to Install Ceph on Raspberry Pi (ARM)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Raspberry Pi, ARM, Linux, Edge, Storage, Embedded

Description: Install Ceph on Raspberry Pi ARM systems using ARM64 packages for edge storage clusters and home lab environments.

---

## Overview

Running Ceph on Raspberry Pi (ARM64) is practical for home labs, edge computing experiments, and learning Ceph without enterprise hardware costs. Raspberry Pi 4 or 5 (4GB+ RAM) with USB SSDs provides enough resources for a functional Ceph cluster. This guide covers ARM64-specific installation on Raspberry Pi OS (64-bit) or Ubuntu Server ARM64.

## Prerequisites

- 3x Raspberry Pi 4 or 5 (4GB or 8GB RAM models)
- 64-bit OS: Raspberry Pi OS (Bookworm 64-bit) or Ubuntu 22.04 ARM64
- USB SSD or NVMe Hat for OSD (not SD cards - too slow and wear out quickly)
- Network switch connecting all Pis
- Static IP addresses or DHCP reservations

## Step 1 - Prepare the Raspberry Pi Nodes

On all Pi nodes:

```bash
# Update system
apt update && apt upgrade -y

# Set hostname
hostnamectl set-hostname pi-ceph-1

# Disable swap
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# Enable IPv4 forwarding
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# Set static IP in /etc/dhcpcd.conf
echo "interface eth0
static ip_address=192.168.1.21/24
static routers=192.168.1.1
static domain_name_servers=1.1.1.1" >> /etc/dhcpcd.conf
```

## Step 2 - Install Ceph for ARM64

Ceph provides ARM64 packages in its Debian repositories:

```bash
# Add Ceph GPG key
curl -fsSL https://download.ceph.com/keys/release.gpg | gpg --dearmor \
  -o /etc/apt/keyrings/ceph.gpg

# Add ARM64 repository
echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/ceph.gpg] https://download.ceph.com/debian-squid/ bookworm main" \
  > /etc/apt/sources.list.d/ceph.list

apt update
apt install -y ceph cephadm
```

## Step 3 - Install Podman for cephadm

```bash
apt install -y podman

# Verify ARM64 container support
podman run --rm arm64v8/alpine uname -m
# Expected output: aarch64
```

## Step 4 - Configure USB/NVMe OSD Devices

Identify the USB SSD device:

```bash
lsblk -o NAME,SIZE,TYPE,TRAN
# Look for sda or sdb with USB transport
```

Wipe the drive:

```bash
wipefs -a /dev/sda
```

## Step 5 - Bootstrap the Cluster

On Pi node 1:

```bash
cephadm bootstrap \
  --mon-ip 192.168.1.21 \
  --single-host-defaults \
  --skip-monitoring-stack \
  --allow-fqdn-hostname
```

The `--skip-monitoring-stack` flag skips Prometheus and Grafana to conserve RAM on the Pi.

## Step 6 - Add Remaining Pi Nodes

```bash
# Copy SSH keys
ssh-copy-id -i /etc/ceph/ceph.pub root@192.168.1.22
ssh-copy-id -i /etc/ceph/ceph.pub root@192.168.1.23

# Add nodes
ceph orch host add pi-ceph-2 192.168.1.22
ceph orch host add pi-ceph-3 192.168.1.23

# Deploy monitors
ceph orch apply mon 3
```

## Step 7 - Deploy OSDs

```bash
# Check USB SSD devices on each node
ceph orch device ls

# Add OSDs from all available drives
ceph orch apply osd --all-available-devices
```

## Step 8 - Verify ARM64 Cluster

```bash
ceph -s
ceph osd tree

# Check it's running ARM64 containers
ceph orch ls | grep running
ceph orch ps | grep active
```

## Performance Expectations

Raspberry Pi 4/5 benchmarks with USB SSD:

```bash
# Quick write benchmark
rados bench -p testpool 30 write --no-cleanup

# Expected throughput: 50-100 MB/s per node
# IOPS: 3000-8000 4K random IOPS depending on SSD
```

## Summary

Ceph runs effectively on Raspberry Pi ARM64 using Debian ARM64 packages from the official Ceph repository. Raspberry Pi 4 and 5 with USB SSDs or NVMe HATs provide enough compute and storage performance for home lab clusters and edge storage experiments. Using `--skip-monitoring-stack` and `--single-host-defaults` reduces resource consumption during initial setup, and the resulting cluster is functionally equivalent to an x86_64 Ceph deployment at smaller scale.
