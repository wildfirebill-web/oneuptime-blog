# How to Configure a GRE Tunnel with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, systemd-networkd, GRE, Tunnel, Networking, Configuration

Description: Configure a GRE tunnel using systemd-networkd .netdev and .network files for persistent tunnel setup that survives reboots.

## Introduction

systemd-networkd creates GRE tunnels via `.netdev` files with `Kind=gre`. The tunnel is automatically created at boot and configured through the corresponding `.network` file. This eliminates the need for startup scripts or manual `ip tunnel` commands.

## Create the GRE Tunnel (.netdev)

```ini
# /etc/systemd/network/40-gre1.netdev
[NetDev]
Name=gre1
Kind=gre

[Tunnel]
Local=203.0.113.1
Remote=203.0.113.2
TTL=64
```

## Configure the GRE Tunnel IP (.network)

```ini
# /etc/systemd/network/40-gre1.network
[Match]
Name=gre1

[Network]
Address=10.10.10.1/30
```

## Add a Static Route Through the Tunnel

```ini
# /etc/systemd/network/40-gre1.network
[Match]
Name=gre1

[Network]
Address=10.10.10.1/30

[Route]
Destination=192.168.50.0/24
Gateway=10.10.10.2
```

## Apply and Verify

```bash
# Restart to create the tunnel
systemctl restart systemd-networkd

# Verify tunnel interface
ip link show gre1
ip addr show gre1

# Check tunnel parameters
ip -d link show gre1

# Test connectivity
ping -c 3 10.10.10.2
```

## Remote Host Configuration (Host B)

```ini
# /etc/systemd/network/40-gre1.netdev
[NetDev]
Name=gre1
Kind=gre

[Tunnel]
Local=203.0.113.2
Remote=203.0.113.1
TTL=64
```

```ini
# /etc/systemd/network/40-gre1.network
[Match]
Name=gre1

[Network]
Address=10.10.10.2/30
```

## GRE with Key (for Multiple Tunnels)

```ini
# /etc/systemd/network/40-gre1.netdev
[NetDev]
Name=gre1
Kind=gre

[Tunnel]
Local=203.0.113.1
Remote=203.0.113.2
Key=1234
TTL=64
```

## Enable IP Forwarding for Tunnel Routing

```bash
# Enable IP forwarding persistently
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-forwarding.conf
sysctl -p /etc/sysctl.d/99-forwarding.conf
```

## Conclusion

GRE tunnels in systemd-networkd require a `.netdev` file with `Kind=gre` and the `[Tunnel]` section specifying `Local` and `Remote` endpoints. Add a `.network` file to configure the tunnel IP and static routes. Changes are applied automatically at boot without startup scripts.
