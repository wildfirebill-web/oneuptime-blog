# How to Understand ARP in Virtualized Environments (VMware, KVM)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, ARP, VMware, KVM, Virtualization

Description: Learn how ARP works in virtual machine environments including VMware vSphere and KVM/libvirt, covering virtual switches, promiscuous mode, and live migration.

## ARP in Virtual Networking

Virtualization platforms create virtual switches (vSwitches) that bridge VMs to the physical network. ARP works similarly to physical networks but with some important considerations.

## VMware vSphere ARP Behavior

### Virtual Switch (vSwitch) Security Settings

VMware vSwitch has security policies that affect ARP:

| Policy | Default | ARP Impact |
|--------|---------|-----------|
| Promiscuous Mode | Reject | VMs only see their own ARP traffic |
| MAC Address Changes | Accept | VMs can change their MAC |
| Forged Transmits | Accept | VMs can send ARP with different MAC |

```
When "Forged Transmits = Reject":
- vSwitch drops ARP packets where source MAC ≠ VM's registered MAC
- Prevents ARP spoofing from inside VMs
```

### Configuring VMware Security Policies

In vSphere Client or via PowerCLI:

```powershell
# PowerCLI: Set security policy on a vSwitch
Get-VirtualSwitch -Name "vSwitch0" | 
    Get-SecurityPolicy | 
    Set-SecurityPolicy -ForgedTransmits $false -MacChanges $false
```

### ARP During vMotion (Live Migration)

When a VM migrates from one host to another via vMotion:

1. VM gets a new MAC address association on the new host
2. VMware sends **gratuitous ARP** on behalf of the VM
3. All switches update their MAC tables
4. Traffic flows to the new host

This is transparent to the VM and completes in milliseconds.

## KVM/libvirt ARP Behavior

### Linux Bridge (Default Mode)

KVM VMs connected via `virbr0` (Linux bridge) share a broadcast domain:

```bash
# View the bridge and connected VMs
brctl show

# Show ARP table for the bridge
ip neigh show dev virbr0

# The bridge acts as a switch — VMs ARP directly to each other
```

### Check VM ARP Traffic

```bash
# Capture ARP on the bridge interface
sudo tcpdump -n -e arp -i virbr0

# Capture on a specific VM's tap interface
sudo tcpdump -n -e arp -i vnet0
```

### MACVTAP Mode

In MACVTAP mode, each VM has its own MAC address directly on the physical NIC:

```bash
# List VM network interfaces (tap/macvtap)
virsh domiflist vm-name

# Check MAC address
virsh domifaddr vm-name
```

### ARP After VM Migration (KVM)

After a KVM live migration, send gratuitous ARP to update network:

```bash
# Inside the VM after migration
arping -A -c 3 -I eth0 192.168.1.100

# Or using virsh network
virsh net-update default
```

## ARP and Promiscuous Mode

For monitoring VMs or virtual network appliances, promiscuous mode allows a VM to see all ARP broadcasts:

```bash
# Enable promiscuous mode on a Linux bridge interface
ip link set virbr0 promisc on

# Verify
ip link show virbr0 | grep PROMISC
```

## Common ARP Issues in Virtualized Environments

| Issue | Cause | Fix |
|-------|-------|-----|
| VM can't reach gateway | vSwitch drops forged transmits | Allow forged transmits |
| ARP after vMotion fails | Gratuitous ARP not propagated | Check portgroup settings |
| Stale ARP after VM restart | Old MAC in physical switch | Flush ARP cache on switch |
| Containers in VM can't ARP | Double NAT or bridge isolation | Check container bridge settings |

## Key Takeaways

- VMware vSwitch security policies (Forged Transmits, MAC Changes) directly affect ARP.
- VMware sends gratuitous ARP automatically after vMotion to update L2 tables.
- KVM/Linux bridge forwards ARP normally between VMs on the same bridge.
- After VM live migration, gratuitous ARP may need to be sent manually in KVM environments.

**Related Reading:**

- [How to Understand Gratuitous ARP and Its Uses](https://oneuptime.com/blog/post/2026-03-20-gratuitous-arp-uses/view)
- [How to Understand ARP in VLAN Environments](https://oneuptime.com/blog/post/2026-03-20-arp-in-vlan-environments/view)
- [How to Debug ARP Issues in Kubernetes Clusters](https://oneuptime.com/blog/post/2026-03-20-debug-arp-kubernetes/view)
