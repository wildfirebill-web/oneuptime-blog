# How to Set Up Ceph RBD Pools in Proxmox

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Proxmox, RBD, Pool, Block Storage, VM

Description: Create and configure Ceph RBD pools optimized for Proxmox VM and container storage, including pool sizing, CRUSH rules, and multiple storage tiers.

---

When using Ceph with Proxmox, proper pool configuration directly impacts VM performance and resilience. This guide covers creating RBD pools specifically for Proxmox workloads, including separate pools for different performance tiers and proper CRUSH rule setup.

## Pool Design Considerations

For a Proxmox-Ceph setup, consider creating separate pools for:
- **VM disk images** - large, high-IOPS, needs good random I/O
- **Container rootfs** - smaller, moderate I/O
- **ISO images** - large files, sequential reads, less critical latency
- **VM backups** (vzdump) - large sequential writes, low priority

## Creating the Core Pools

On the Ceph admin node:

```bash
# VM disk images pool
ceph osd pool create pve-vms 128
rbd pool init pve-vms
ceph osd pool set pve-vms size 3
ceph osd pool set pve-vms min_size 2
ceph osd pool set pve-vms pg_autoscale_mode on

# Container storage pool
ceph osd pool create pve-ct 64
rbd pool init pve-ct
ceph osd pool set pve-ct size 3
ceph osd pool set pve-ct pg_autoscale_mode on

# ISO/template storage (lower replication, less critical)
ceph osd pool create pve-iso 32
rbd pool init pve-iso
ceph osd pool set pve-iso size 2
```

## Creating CRUSH Rules for Tiered Storage

If you have mixed OSD devices (SSD + HDD), create separate CRUSH rules:

```bash
# List available device classes
ceph osd crush class ls
# Output: hdd, ssd

# Create rules for SSD and HDD OSDs
ceph osd crush rule create-replicated pve-ssd-rule default host ssd
ceph osd crush rule create-replicated pve-hdd-rule default host hdd

# Assign VM pool to SSD (for performance)
ceph osd pool set pve-vms crush_rule pve-ssd-rule

# Assign container and ISO pools to HDD (cost efficiency)
ceph osd pool set pve-ct crush_rule pve-hdd-rule
ceph osd pool set pve-iso crush_rule pve-hdd-rule
```

## Configuring Pool Application Tags

```bash
# Tag all pools with the rbd application
for pool in pve-vms pve-ct pve-iso; do
  ceph osd pool application enable ${pool} rbd
done

# Verify
ceph osd pool application get pve-vms
```

## Adding Multiple Pools to Proxmox

```bash
# On a Proxmox node
# Add the VM pool
pvesm add rbd pve-ceph-vms \
  --monhost "mon1:6789,mon2:6789,mon3:6789" \
  --pool pve-vms \
  --username proxmox \
  --keyring /etc/ceph/ceph.client.proxmox.keyring \
  --content images

# Add the container pool
pvesm add rbd pve-ceph-ct \
  --monhost "mon1:6789,mon2:6789,mon3:6789" \
  --pool pve-ct \
  --username proxmox \
  --keyring /etc/ceph/ceph.client.proxmox.keyring \
  --content rootdir

# Verify both are online
pvesm status | grep pve-ceph
```

## Monitoring Pool Usage

```bash
# Check pool usage and object counts
ceph df detail | grep -E "pve-|USED|NAME"

# Check PG distribution
ceph osd pool stats pve-vms
ceph osd pool stats pve-ct

# Watch I/O on the pools in real time
ceph iostat 2 | head -30
```

## Setting Pool Quotas

```bash
# Limit pve-vms to 10 TB
ceph osd pool set-quota pve-vms max_bytes $((10 * 1024**4))

# Check quotas
ceph osd dump | grep -A2 "pool.*pve-vms"
```

## Summary

Setting up Ceph RBD pools for Proxmox requires creating separate pools for VMs, containers, and ISOs with appropriate replication levels and CRUSH rules. Assign high-performance SSD pools to VM disk images for best latency and use HDD pools for containers and ISO storage where cost efficiency matters more. Register each pool as a separate Proxmox storage definition via `pvesm add rbd` to make them independently selectable when creating VMs and containers. Use PG autoscaling to let Ceph manage PG counts as data grows.
