# How to Fix Missing CPU/Memory Stats in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Stats, Docker, Cgroups, Raspberry Pi, LXC

Description: Learn how to fix missing CPU and memory statistics in Portainer, including enabling cgroup memory accounting, fixing LXC container limitations, and kernel parameter issues.

---

The container statistics panel in Portainer shows "N/A" or missing charts when the Docker host cannot collect cgroup metrics. This is common on Raspberry Pi, LXC containers, and some minimal Linux installations.

## Step 1: Check Docker Stats Directly

First verify the issue is at the Docker level, not just in Portainer:

```bash
# Test if Docker can collect stats

docker stats --no-stream

# If all values show 0 or N/A for memory, the issue is in cgroups
# If stats work in CLI but not in Portainer, it's a Portainer connectivity issue
```

## Step 2: Enable cgroup Memory Accounting (Raspberry Pi / ARM)

Raspberry Pi OS disables memory cgroup by default. Edit the kernel cmdline:

```bash
# Edit the boot command line
sudo nano /boot/cmdline.txt

# Add these parameters to the existing single line (do NOT add a new line):
cgroup_enable=memory cgroup_memory=1 cgroup_enable=cpuset

# Reboot
sudo reboot
```

After reboot, verify:

```bash
# Check that memory cgroup is enabled
cat /proc/cgroups | grep memory
# Should show: memory  0  XX  1  (the last 1 means enabled)
```

## Step 3: Fix Missing Stats in LXC Containers

If Docker runs inside an LXC container (Proxmox), enable cgroup nesting:

In the Proxmox LXC configuration file (e.g., `/etc/pve/lxc/100.conf`):

```bash
# Add these lines to enable cgroup nesting
lxc.cgroup2.memory.limit_in_bytes = max
features: nesting=1,keyctl=1
```

Restart the LXC container after editing.

## Step 4: Verify Docker Daemon cgroup Driver

```bash
# Check the Docker cgroup driver
docker info | grep "Cgroup Driver"
# Should show: Cgroup Driver: cgroupfs or systemd

# If blank, configure it explicitly in /etc/docker/daemon.json
sudo nano /etc/docker/daemon.json
```

```json
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"]
}
```

```bash
sudo systemctl restart docker
```

## Step 5: Check Agent Permissions

If using the Portainer Agent, ensure it has access to `/proc`:

```bash
# The agent container needs access to the host /proc filesystem
docker run ... \
  -v /proc:/host/proc:ro \
  portainer/agent:latest
```
