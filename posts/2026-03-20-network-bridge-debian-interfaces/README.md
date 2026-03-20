# How to Create a Network Bridge on Debian Using /etc/network/interfaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Bridge, Debian, /etc/network/interfaces, Linux, KVM, Virtual Networking, IPv4

Description: Learn how to create a Linux network bridge on Debian using /etc/network/interfaces for VM networking with KVM, connecting VMs directly to the physical network via bridging.

---

A network bridge connects two or more network interfaces at Layer 2, allowing virtual machines to appear as physical hosts on the network. This is the standard approach for KVM VM networking.

## Installing Bridge Tools

```bash
apt install bridge-utils -y
```

## Basic Bridge Configuration

```bash
# /etc/network/interfaces

# Physical interface: no IP, added to bridge

auto eth0
iface eth0 inet manual

# Bridge: has the IP
auto br0
iface br0 inet static
  address 192.168.1.10
  netmask 255.255.255.0
  gateway 192.168.1.1
  dns-nameservers 8.8.8.8

  # Bridge options
  bridge_ports eth0          # Physical interface(s) to bridge
  bridge_stp on              # Enable STP to prevent loops
  bridge_fd 0                # Forwarding delay (0 for VM setups; normally 15s)
  bridge_maxwait 0
```

## Bridge with DHCP

```bash
# /etc/network/interfaces
auto eth0
iface eth0 inet manual

auto br0
iface br0 inet dhcp
  bridge_ports eth0
  bridge_stp on
  bridge_fd 0
```

## Applying the Configuration

```bash
# Restart networking
systemctl restart networking

# Or bring up manually
ifup br0

# Verify bridge
brctl show
# Output:
# bridge name  bridge id           STP enabled  interfaces
# br0          8000.aabbccddeeff   yes          eth0

ip addr show br0
```

## KVM/QEMU VM on Bridge

```xml
<!-- VM network interface (libvirt XML) -->
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
</interface>
```

## Adding Multiple Interfaces to Bridge

```bash
# /etc/network/interfaces - bridge across two physical NICs
auto br0
iface br0 inet static
  address 192.168.1.10
  netmask 255.255.255.0
  bridge_ports eth0 eth1    # Both ports bridged together
  bridge_stp on
  bridge_fd 0
```

## Troubleshooting

```bash
# Check bridge state
brctl show
brctl showstp br0

# Check STP port states (should be FORWARDING, not BLOCKING)
brctl showstp br0 | grep state

# Show all forwarding database entries (learned MACs)
brctl showmacs br0

# Check bridge interface in kernel
ip -d link show br0
```

## Key Takeaways

- Set `bridge_fd 0` for VM hosting to eliminate the 15-second forwarding delay at startup.
- The physical interface (`eth0`) has no IP; the bridge interface (`br0`) holds the host IP.
- Enable STP (`bridge_stp on`) to prevent loops when multiple bridge paths exist.
- `brctl show` and `brctl showstp` are the primary tools for inspecting bridge state.
