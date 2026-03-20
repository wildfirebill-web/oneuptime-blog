# How to Create a .link File for Interface Renaming in systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: systemd-networkd, .link file, Interface Renaming, udev, Linux, Predictable Names, Networking

Description: Learn how to create systemd-networkd .link files to rename network interfaces from predictable names (like ens3) to custom names (like eth0 or wan0) based on MAC address or other properties.

---

`.link` files in `/etc/systemd/network/` control interface renaming, driver settings, and link parameters. They are processed by `udev` during device enumeration.

## Why Rename Interfaces

- Consistent naming across machines (always `eth0` regardless of hardware)
- Meaningful names for clarity (`wan0`, `lan0`, `mgmt0`)
- Legacy application compatibility that expects `eth0`

## Basic Interface Renaming by MAC Address

```ini
# /etc/systemd/network/10-rename-eth0.link
[Match]
MACAddress=aa:bb:cc:dd:ee:01

[Link]
Name=eth0
```

```bash
# Apply: reload udev rules or reboot
udevadm control --reload
# The rename takes effect on next hotplug or reboot

# Verify after reboot
ip link show eth0
```

## Renaming by PCI Bus Path

```ini
# /etc/systemd/network/10-wan.link
[Match]
Path=pci-0000:02:00.0   # PCI path from: udevadm info /sys/class/net/ens3

[Link]
Name=wan0
```

## Finding Interface Properties for Matching

```bash
# Find MAC address
ip link show ens3 | grep "link/ether"

# Find PCI path
udevadm info /sys/class/net/ens3 | grep -E "ID_PATH|ID_NET_NAME"

# Find driver
ethtool -i ens3 | grep driver
```

## Setting Link Parameters in .link Files

```ini
# /etc/systemd/network/10-eth0.link
[Match]
MACAddress=aa:bb:cc:dd:ee:01

[Link]
Name=eth0

# Additional link settings:
MTUBytes=9000           # Set MTU
WakeOnLan=magic         # Enable Wake-on-LAN
Duplex=full             # Force duplex
Speed=1000              # Force speed (Mbps)
AutoNegotiation=yes     # Enable autoneg
```

## Disabling Predictable Interface Names

To revert to kernel naming (ethX):

```bash
# /etc/systemd/network/10-ethname.link
[Match]
OriginalName=*         # Match all interfaces

[Link]
NamePolicy=kernel      # Use kernel-assigned names only
```

Or via kernel parameter:
```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
update-grub
```

## Verifying .link File Processing

```bash
# Check which .link file matched an interface
udevadm test-builtin net_setup_link /sys/class/net/eth0

# View udev events for interface
udevadm monitor --property | grep -A10 "net"
```

## Key Takeaways

- `.link` files in `/etc/systemd/network/` rename interfaces at udev time, before network configuration.
- Match by `MACAddress` for hardware-specific renaming; use `Path` for PCI-location-based naming.
- The `Name=` directive in the `[Link]` section sets the new interface name.
- Changes take effect on reboot or when the device is hotplugged; test with `udevadm test-builtin`.
