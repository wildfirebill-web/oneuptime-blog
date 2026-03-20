# How to Set the MTU on an Interface with Netplan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Netplan, MTU, Linux, Networking, Ubuntu, Performance

Description: Set a custom Maximum Transmission Unit (MTU) on a network interface using Netplan's YAML configuration to optimize throughput or avoid fragmentation.

## Introduction

The Maximum Transmission Unit (MTU) defines the largest packet size that can be transmitted without fragmentation. The default Ethernet MTU is 1500 bytes, but you may need to change it for jumbo frames (9000 bytes) or VPN tunnels (1400 bytes or less). Netplan lets you set the MTU declaratively in YAML.

## When to Change MTU

- **Jumbo frames**: Increase to 9000 bytes on 10GbE storage networks to reduce CPU overhead
- **VPN/tunnel interfaces**: Decrease to 1400–1450 to avoid fragmentation inside tunnels
- **Cloud instances**: Match the MTU of the underlying network fabric (e.g., 1450 on some cloud providers)

## Setting MTU with Netplan

Edit your Netplan configuration file (e.g., `/etc/netplan/01-netcfg.yaml`):

```yaml
# /etc/netplan/01-netcfg.yaml

network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 192.168.1.10/24
      routes:
        - to: default
          via: 192.168.1.1
      mtu: 9000    # Enable jumbo frames for this interface
```

For a VPN or WireGuard-style tunnel with a reduced MTU:

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 10.0.0.5/24
      mtu: 1420    # Reduced MTU to fit inside WireGuard overhead
```

## Applying and Verifying

```bash
# Safely test the configuration (auto-reverts after 120s if not confirmed)
sudo netplan try

# Apply permanently
sudo netplan apply

# Verify the MTU was set correctly
ip link show eth0
```

Look for `mtu 9000` (or your chosen value) in the output.

## Testing End-to-End MTU

After changing the MTU, verify that large packets traverse the path without fragmentation:

```bash
# Send a ping with a specific packet size (subtract 28 bytes for ICMP+IP headers)
ping -M do -s 8972 192.168.1.1
```

If you receive a response, jumbo frames are working end-to-end. If the ping fails, a device in the path does not support the MTU.

## Important Notes

- All switches and NICs in the path must support the same MTU for jumbo frames to work
- Mismatched MTU causes silent packet drops and poor performance - not always an obvious error
- Use `ip link set eth0 mtu 9000` for a temporary, non-persistent change to test before editing Netplan

## Conclusion

Netplan's `mtu` field gives you a persistent, readable way to configure interface MTU. Always test end-to-end before deploying jumbo frames in production.
