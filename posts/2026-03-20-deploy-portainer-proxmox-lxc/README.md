# How to Deploy Portainer in a Proxmox LXC Container

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Proxmox, LXC, Docker, Self-Hosted, Virtualization

Description: Learn how to create an LXC container in Proxmox VE and deploy Portainer CE inside it for Docker container management.

---

Proxmox LXC (Linux Containers) provide lightweight virtualization with less overhead than full VMs. Running Docker and Portainer inside an LXC container is more efficient than a VM and works well for home lab environments.

---

## Create an Unprivileged LXC Container

Using Proxmox `pct` CLI on the Proxmox host:

```bash
# Download Ubuntu 22.04 template if not available

pveam update
pveam download local ubuntu-22.04-standard_22.04-1_amd64.tar.zst

# Create the container
pct create 101 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst   --hostname portainer   --memory 2048   --cores 2   --net0 name=eth0,bridge=vmbr0,ip=dhcp   --storage local-lvm   --rootfs local-lvm:8   --unprivileged 1   --features nesting=1  # Required for Docker in LXC

pct start 101
```

---

## Configure the LXC for Docker

```bash
# On the Proxmox host, edit the LXC config
cat >> /etc/pve/lxc/101.conf <<'EOF'
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
EOF

# Restart the container
pct restart 101
```

---

## Install Docker Inside the LXC

```bash
# Enter the container
pct exec 101 -- bash

# Install Docker
apt-get update
curl -fsSL https://get.docker.com | sh

# Verify
docker run hello-world
```

---

## Deploy Portainer Inside the LXC

```bash
pct exec 101 -- bash -c '
  docker volume create portainer_data
  docker run -d     --name portainer     --restart=always     -p 9000:9000     -p 9443:9443     -v /var/run/docker.sock:/var/run/docker.sock     -v portainer_data:/data     portainer/portainer-ce:latest
'
```

---

## Autostart with Proxmox

```bash
# Enable container autostart
pct set 101 --onboot 1
```

---

## Summary

Create a Proxmox LXC container with `--features nesting=1` to enable Docker. Add `lxc.apparmor.profile: unconfined` to the container config file for full Docker functionality. Install Docker inside the container and run Portainer with `--restart=always`. Enable `--onboot 1` to start the container automatically when Proxmox boots.
