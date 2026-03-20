# How to Configure IPv6 for VM Migration (vMotion/Live Migration)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, VM Migration, VMotion, Live Migration, VMware, Hyper-V, KVM

Description: Configure IPv6 for virtual machine live migration in VMware vMotion, Hyper-V Live Migration, and KVM QEMU migration, ensuring the migration network uses IPv6 and VMs retain connectivity after...

## Introduction

VM live migration moves running virtual machines between hypervisor hosts with minimal downtime. The migration network (vMotion in VMware, Live Migration in Hyper-V, QEMU migration in KVM) can use IPv6 for the migration data transfer, separating it from VM traffic. After migration, VMs retain their IPv6 addresses because NDP updates (gratuitous NDP) notify the network of the new MAC-to-IP mapping.

## VMware vMotion over IPv6

```bash
# Configure vMotion VMkernel adapter for IPv6

# Step 1: Add IPv6 address to vMotion VMkernel adapter (esxcli)

esxcli network ip interface ipv6 address add \
    --interface-name vmk1 \
    --ip 2001:db8:vmotion::10 \
    --prefix-length 64

# Step 2: Enable vMotion traffic on the VMkernel adapter
# via vSphere Client: Host → Configure → Networking → VMkernel Adapters
# → Edit vmk1 → IPv6 settings → Add address
# → Port Services: check "vMotion"

# Step 3: Verify vMotion is using IPv6
# vSphere Client → Monitor → vMotion → Migration History
# Shows source and destination VMkernel addresses

# Step 4: Configure vMotion network CIDR on all hosts
# All hosts in the cluster must have vMotion VMkernel IPv6 addresses
# in the same /64 or with reachable routes
```

## Hyper-V Live Migration over IPv6

```powershell
# Configure Live Migration to use specific IPv6 network

# Enable Live Migration
Enable-VMMigration -ComputerName "hyper-v-host1"

# Add IPv6 network for Live Migration
Add-VMMigrationNetwork -ComputerName "hyper-v-host1" `
    -Subnet "2001:db8:livemig::/64" `
    -Priority 1

# Verify migration networks
Get-VMMigrationNetwork -ComputerName "hyper-v-host1"
# Shows: 2001:db8:livemig::/64

# Test Live Migration over IPv6 to another host
Move-VM -Name "MyVM" `
    -DestinationHost "hyper-v-host2" `
    -DestinationStoragePath "C:\VMs\MyVM"

# For storage Live Migration over IPv6:
Move-VM -Name "MyVM" `
    -DestinationHost "[2001:db8::20]" `
    -IncludeStorage `
    -DestinationStoragePath "C:\VMs\MyVM"
```

## KVM/QEMU Live Migration over IPv6

```bash
# QEMU supports live migration over IPv6 natively

# On destination host: listen for incoming migration
qemu-system-x86_64 \
    -drive file=myvm.qcow2,format=qcow2 \
    -m 2048 \
    -monitor stdio \
    -incoming "tcp:[2001:db8::20]:4444"
# [2001:db8::20] = destination host's IPv6

# On source host: initiate migration to IPv6 destination
virsh migrate --live --verbose myvm \
    "qemu+tcp://[2001:db8::20]/system"
# or
virsh migrate --live myvm \
    "qemu+ssh://root@[2001:db8::20]/system"

# QEMU monitor command:
# migrate tcp:[2001:db8::20]:4444
```

```bash
# libvirt migration with IPv6 (requires libvirt on both hosts)

# Configure libvirt to use IPv6 for migration transport
# /etc/libvirt/libvirtd.conf
listen_addr = "2001:db8::10"   # Listen on IPv6

# Migrate VM between hosts using IPv6
virsh migrate --live --p2p --tunnelled myvm \
    "qemu+tcp://[2001:db8::20]/system"
```

## After Migration: IPv6 Address Continuity

```bash
# After live migration, the VM retains its IPv6 address
# The hypervisor sends a gratuitous NDP to update neighbor caches

# Verify VM still has its IPv6 after migration
virsh domifaddr myvm  # KVM
# or
qm guest exec 101 ip -6 addr show  # Proxmox

# If neighbor cache stale after migration, trigger NDP update
# Inside the VM (Linux):
arping6 -c 3 -i eth0 2001:db8::100   # If arping6 available
# Or:
ndsend 2001:db8::100 eth0            # Explicit NDP advertisement

# On switches: neighbor cache should update within seconds
# Check on a router:
# show ipv6 neighbors | include 2001:db8::100
```

## Migration Network Firewall for IPv6

```bash
# Allow QEMU migration port over IPv6
ip6tables -A INPUT -p tcp --dport 49152:49215 \
    -s 2001:db8:vmotion::/64 -j ACCEPT

# Allow libvirt daemon communication
ip6tables -A INPUT -p tcp --dport 16509 \
    -s 2001:db8:vmotion::/64 -j ACCEPT

# Allow SSH for libvirt migration tunneling
ip6tables -A INPUT -p tcp --dport 22 \
    -s 2001:db8:vmotion::/64 -j ACCEPT

ip6tables-save > /etc/ip6tables/rules.v6
```

## Verify IPv6 Migration

```bash
# KVM: test migration reachability
virsh -c "qemu+tcp://[2001:db8::20]/system" list --all

# VMware: check vMotion network connectivity
esxcli network diag ping \
    --host 2001:db8:vmotion::20 \
    --netstack vmotion \
    --ipv6

# Hyper-V: test Live Migration
Test-VMMigration -Name "MyVM" -DestinationHost "[2001:db8::20]"
```

## Conclusion

VM live migration over IPv6 is supported natively by VMware vMotion (configure vMotion VMkernel with IPv6 addresses), Hyper-V Live Migration (use `Add-VMMigrationNetwork` with IPv6 CIDR), and KVM QEMU migration (use bracket notation for IPv6 destination in `virsh migrate`). After migration, VMs retain their IPv6 addresses - the hypervisor sends gratuitous NDP to update neighbor caches on connected switches. Use a dedicated IPv6 network segment for migration traffic to isolate it from VM data traffic and simplify firewall rules. The migration network IPv6 prefix should be reachable between all hosts in the cluster without going through the production network.
