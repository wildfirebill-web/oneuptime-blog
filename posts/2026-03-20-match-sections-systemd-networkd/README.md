# How to Use Match Sections to Target Specific Interfaces in systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: systemd-networkd, Match Section, Interface Selection, Linux, .network, .link, Networking

Description: Learn how to use [Match] sections in systemd-networkd .network and .link files to precisely target specific network interfaces using name, MAC address, driver, and other properties.

---

The `[Match]` section determines which interfaces a `.network` or `.link` file applies to. Precise matching prevents configurations from applying to unintended interfaces.

## Match by Interface Name

```ini
# /etc/systemd/network/eth0.network
[Match]
Name=eth0          # Exact name match

# Wildcard
[Match]
Name=eth*          # Match eth0, eth1, eth2...

# Multiple names
[Match]
Name=eth0 eth1     # Match eth0 or eth1
```

## Match by MAC Address

```ini
[Match]
MACAddress=aa:bb:cc:dd:ee:01   # Exact MAC match

# Useful for servers where interface names may vary
```

## Match by Driver

```ini
[Match]
Driver=virtio_net   # All virtio network interfaces (VMs)

[Match]
Driver=igb          # All Intel igb NIC interfaces
```

## Match by PCI Path

```ini
[Match]
Path=pci-0000:02:00.0   # Specific PCI slot

# Find path: udevadm info /sys/class/net/eth0 | grep ID_PATH
```

## Match by Interface Type

```ini
[Match]
Type=ether          # All Ethernet interfaces

[Match]
Type=vlan           # All VLAN interfaces

[Match]
Type=bond           # All bond interfaces

# Types: ether, wlan, loopback, vlan, bond, bridge, tun, tap, dummy, ...
```

## Match by Virtualization

```ini
[Match]
Virtualization=vm   # Only match inside a VM
# Virtualization=container  # Only in containers
# Virtualization=no         # Only on bare metal
```

## Match by Host

```ini
[Match]
Host=webserver-01   # Only on this specific hostname
```

## Combining Match Criteria (AND logic)

```ini
# All criteria must match (logical AND)
[Match]
Name=eth*
Driver=igb
MACAddress=aa:bb:cc:*:*:*
```

## Match Precedence

```
Files are processed in lexicographic order (10- before 20- before 99-)
More specific files should have lower numbers to be processed first
The FIRST matching file wins (for .link files)
For .network files, all matching files are merged
```

## Example: Different Config for Physical vs. Virtual

```ini
# /etc/systemd/network/10-vm.network — VMs (virtio)
[Match]
Driver=virtio_net
Virtualization=vm

[Network]
DHCP=yes

# /etc/systemd/network/20-physical.network — Physical servers
[Match]
Type=ether
Virtualization=no

[Network]
Address=10.0.0.5/24
Gateway=10.0.0.1
```

## Checking Which File Matched

```bash
# Show which network file is applied to an interface
networkctl status eth0 | grep "Network File"
# Network File: /etc/systemd/network/10-eth0.network

# Test match without applying
networkd-manager --test-match=eth0
```

## Key Takeaways

- `[Match]` sections use AND logic: all specified criteria must match.
- Match by `MACAddress` for hardware-specific configs; by `Name=*` glob for categories of interfaces.
- Files are processed lexicographically; lower-numbered files take precedence.
- Use `networkctl status <iface>` to confirm which `.network` file was applied to an interface.
