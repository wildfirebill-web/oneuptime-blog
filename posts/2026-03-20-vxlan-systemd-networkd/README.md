# How to Configure VXLAN with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: VXLAN, systemd-networkd, Linux, Overlay Networking, IPv4, .netdev, Persistent

Description: Learn how to configure VXLAN interfaces persistently using systemd-networkd .netdev and .network files for declarative overlay network configuration.

---

systemd-networkd supports VXLAN natively through `.netdev` files, enabling persistent VXLAN configuration that survives reboots without additional scripts.

## Creating a VXLAN Interface with systemd-networkd

### Step 1: Define the VXLAN Device

```ini
# /etc/systemd/network/vxlan10.netdev

[NetDev]
Name=vxlan10
Kind=vxlan

[VXLAN]
VNI=10                         # VXLAN Network Identifier
Remote=10.0.0.2                # Remote VTEP (for point-to-point)
Local=10.0.0.1                 # Local VTEP IP
DestinationPort=4789           # Standard VXLAN UDP port
Independent=false              # Bind to underlay interface
MacLearning=yes                # Enable MAC learning
```

### Step 2: Configure the VXLAN Network

```ini
# /etc/systemd/network/vxlan10.network
[Match]
Name=vxlan10

[Network]
Address=192.168.100.1/24

# Optionally attach to a bridge
# Bridge=br-vxlan
```

### Step 3: Apply

```bash
networkctl reload
networkctl status vxlan10
```

## VXLAN with Multicast Discovery

```ini
# /etc/systemd/network/vxlan10.netdev
[NetDev]
Name=vxlan10
Kind=vxlan

[VXLAN]
VNI=10
Group=239.1.1.10               # Multicast group for BUM traffic
Local=10.0.0.1
DestinationPort=4789
```

## VXLAN Attached to a Bridge

```ini
# /etc/systemd/network/br-vxlan.netdev
[NetDev]
Name=br-vxlan
Kind=bridge

[Bridge]
VLANFiltering=no
```

```ini
# /etc/systemd/network/br-vxlan.network
[Match]
Name=br-vxlan

[Network]
Address=192.168.100.1/24
```

```ini
# /etc/systemd/network/vxlan10.netdev
[NetDev]
Name=vxlan10
Kind=vxlan

[VXLAN]
VNI=10
Local=10.0.0.1
DestinationPort=4789
MacLearning=yes
```

```ini
# /etc/systemd/network/vxlan10.network
[Match]
Name=vxlan10

[Network]
Bridge=br-vxlan    # Attach VXLAN to the bridge
```

## Available VXLAN Options

```ini
[VXLAN]
VNI=10                  # 1-16777215
Local=10.0.0.1          # Local VTEP IP
Remote=10.0.0.2         # Remote VTEP (or use Group for multicast)
Group=239.1.1.10        # Multicast group
DestinationPort=4789    # UDP port (default 4789)
MacLearning=yes         # Enable/disable MAC learning
TTL=64                  # Outer packet TTL
TOS=0                   # Type of Service
FlowLabel=0             # IPv6 flow label
MaximumFDBEntries=4096  # Maximum FDB size
ReduceARPProxy=no       # ARP proxy suppression
UDPChecksum=no          # UDP checksum
UDP6ZeroChecksumTx=no
UDP6ZeroChecksumRx=no
```

## Verifying Configuration

```bash
networkctl list | grep vxlan
networkctl status vxlan10
ip -d link show vxlan10
bridge fdb show dev vxlan10
```

## Key Takeaways

- systemd-networkd creates VXLAN interfaces via `.netdev` files with a `[VXLAN]` section specifying VNI, local/remote IPs, and port.
- For multi-VTEP deployments, use `Group` (multicast) instead of `Remote` (unicast).
- Attach the VXLAN interface to a bridge by setting `Bridge=<bridge-name>` in the `.network` file.
- Run `networkctl reload` to apply changes; use `networkctl status` to verify.
