# How to Fix Missing CPU/Memory Stats in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Monitoring, Performance

Description: Resolve missing or zero CPU and memory statistics in Portainer's container dashboard, including cgroup configuration, Docker stats API issues, and kernel compatibility fixes.

## Introduction

Portainer displays CPU and memory usage by polling the Docker stats API. When stats show as zero, "N/A", or don't update, the issue is almost always a Linux kernel cgroup configuration problem, Docker daemon settings, or a missing permission. This guide walks through all the fixes.

## Step 1: Verify Docker Stats Work from CLI

```bash
# Test if Docker stats work at all
docker stats --no-stream

# If all values are 0 or N/A, the issue is at the Docker/kernel level
# If values are present, the issue is Portainer-specific
```

## Step 2: Enable cgroup Memory Accounting (Most Common Fix)

On many Linux distributions, memory accounting is disabled by default to save overhead:

```bash
# Check current cgroup status
cat /sys/fs/cgroup/memory/memory.stat 2>/dev/null | head -5

# If the file doesn't exist, memory cgroups are disabled
# Check kernel boot parameters
grep -i cgroup /proc/cmdline

# For Ubuntu/Debian with GRUB, enable memory cgroup:
sudo sed -i 's/GRUB_CMDLINE_LINUX="/GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1 /' /etc/default/grub

# Or add directly:
sudo nano /etc/default/grub
# Change:
# GRUB_CMDLINE_LINUX=""
# To:
# GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"

# Update GRUB and reboot
sudo update-grub
sudo reboot
```

## Step 3: Check cgroup v1 vs v2

Some distributions use cgroup v2 (unified hierarchy), which Docker supports but may need configuration:

```bash
# Check cgroup version
mount | grep cgroup

# For cgroup v2 detection:
ls /sys/fs/cgroup/cgroup.controllers 2>/dev/null && echo "cgroup v2" || echo "cgroup v1"

# Configure Docker for cgroup v2
cat > /etc/docker/daemon.json << 'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

sudo systemctl restart docker
docker restart portainer
```

## Step 4: Fix on Raspberry Pi / ARM Devices

Raspberry Pi OS requires specific boot configuration for cgroup memory:

```bash
# Edit boot config
sudo nano /boot/cmdline.txt

# Add to the end of the single line (don't add a new line):
# cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory

# The line should look like:
# console=serial0,115200 console=tty1 root=PARTUUID=... cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory

sudo reboot
```

## Step 5: Check Docker Daemon Configuration

```bash
# Verify Docker daemon stats settings
cat /etc/docker/daemon.json

# Some flags disable stats collection
# Ensure these are NOT set:
# "no-new-privileges": true  (may affect stats in some versions)

# Restart Docker after any changes
sudo systemctl restart docker
```

## Step 6: Fix on LXC/Proxmox Containers

Running Docker inside an LXC container requires specific LXC configuration:

```bash
# On the Proxmox host, edit the LXC container configuration
# /etc/pve/lxc/<container-id>.conf

# Add these lines:
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
features: nesting=1,keyctl=1
```

Or use the Proxmox UI:
1. Container → Options → Features
2. Enable **Nesting** and **keyctl**

## Step 7: Verify Portainer Has Access to Stats API

```bash
# Test the Docker stats API directly
curl --unix-socket /var/run/docker.sock \
  "http://localhost/v1.44/containers/<container-id>/stats?stream=false" | jq .

# Check if cpu_stats and memory_stats have real values
# If both show 0, it's a cgroup issue
# If they have values, the issue is how Portainer reads them
```

## Step 8: Check Portainer Agent Configuration

When using the Portainer Agent, stats are fetched by the agent:

```bash
# Check agent logs for stats errors
docker logs portainer-agent 2>&1 | grep -i "stats\|cgroup\|memory"

# Restart the agent
docker restart portainer-agent
```

## Step 9: Fix for Specific Kernel Versions

Some kernel versions have cgroup accounting bugs:

```bash
# Check kernel version
uname -r

# Update the kernel if significantly outdated
sudo apt-get update && sudo apt-get dist-upgrade

# On Ubuntu, install HWE kernel for better hardware support
sudo apt-get install linux-generic-hwe-22.04
sudo reboot
```

## Step 10: Verify with cAdvisor

Use cAdvisor to verify metrics are available outside of Portainer:

```bash
# Deploy cAdvisor
docker run -d \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --name=cadvisor \
  gcr.io/cadvisor/cadvisor:latest

# Access at http://your-host:8080
# If cAdvisor also shows zero stats, it's a kernel/cgroup issue
# If cAdvisor shows stats but Portainer doesn't, update Portainer
```

## Conclusion

Missing CPU/memory stats in Portainer are almost always caused by cgroup memory accounting being disabled in the kernel boot configuration. The fix is to add `cgroup_enable=memory swapaccount=1` to the kernel command line and reboot. On Raspberry Pi, add `cgroup_memory=1 cgroup_enable=memory` to `/boot/cmdline.txt`. If Docker CLI stats work but Portainer doesn't show them, update Portainer to the latest version.
