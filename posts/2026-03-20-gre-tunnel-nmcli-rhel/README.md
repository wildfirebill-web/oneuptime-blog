# How to Create a GRE Tunnel Using nmcli on RHEL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GRE, Tunnel, nmcli, RHEL, NetworkManager, IPv4, Networking

Description: Learn how to create and manage a GRE tunnel on RHEL/CentOS using nmcli (NetworkManager CLI) for persistent tunnel configuration that survives reboots.

---

GRE (Generic Routing Encapsulation) tunnels encapsulate network packets to create virtual point-to-point links between hosts. On RHEL, nmcli provides persistent tunnel management through NetworkManager.

## Creating a GRE Tunnel with nmcli

```bash
# Create a GRE tunnel connection

nmcli connection add type ip-tunnel \
  ifname gre1 \
  con-name gre-tunnel-1 \
  ip-tunnel.mode gre \
  ip-tunnel.local 10.0.0.1 \        # This host's tunnel source IP
  ip-tunnel.remote 10.0.0.2         # Remote tunnel endpoint IP

# Assign IP to the tunnel interface
nmcli connection modify gre-tunnel-1 \
  ipv4.method manual \
  ipv4.addresses "172.16.1.1/30"

# Activate the tunnel
nmcli connection up gre-tunnel-1
```

## Verify the GRE Tunnel

```bash
# Show tunnel interface
ip -d link show gre1
# gre: remote 10.0.0.2 local 10.0.0.1 dev eth0 ttl inherit

# Check IP assignment
ip addr show gre1

# Ping the remote tunnel endpoint
ping 172.16.1.2

# Show routing through tunnel
ip route show dev gre1
```

## Adding Routes Through the GRE Tunnel

```bash
# Route traffic to remote subnet through GRE tunnel
nmcli connection modify gre-tunnel-1 \
  +ipv4.routes "192.168.2.0/24 172.16.1.2"

nmcli connection up gre-tunnel-1

# Verify
ip route show | grep 192.168.2.0
```

## GRE Tunnel Configuration File (for reference)

NetworkManager stores the connection in `/etc/NetworkManager/system-connections/`:

```ini
# /etc/NetworkManager/system-connections/gre-tunnel-1.nmconnection
[connection]
id=gre-tunnel-1
type=ip-tunnel
interface-name=gre1

[ip-tunnel]
mode=3       # 3 = GRE
local=10.0.0.1
remote=10.0.0.2
ttl=255

[ipv4]
method=manual
address1=172.16.1.1/30
route1=192.168.2.0/24,172.16.1.2

[ipv6]
method=ignore
```

## Remote Side Configuration

```bash
# On the remote host (10.0.0.2):
nmcli connection add type ip-tunnel \
  ifname gre1 \
  con-name gre-tunnel-1 \
  ip-tunnel.mode gre \
  ip-tunnel.local 10.0.0.2 \
  ip-tunnel.remote 10.0.0.1

nmcli connection modify gre-tunnel-1 \
  ipv4.method manual \
  ipv4.addresses "172.16.1.2/30"

nmcli connection up gre-tunnel-1
```

## Managing the Tunnel

```bash
# Bring down tunnel
nmcli connection down gre-tunnel-1

# Delete tunnel
nmcli connection delete gre-tunnel-1

# Show all tunnels
nmcli connection show | grep tunnel
```

## Key Takeaways

- Use `nmcli connection add type ip-tunnel` with `ip-tunnel.mode gre` to create persistent GRE tunnels on RHEL.
- Set `ip-tunnel.local` to the outbound interface IP and `ip-tunnel.remote` to the peer's IP.
- Add static routes with `+ipv4.routes` to direct traffic through the tunnel.
- nmcli stores connections persistently; tunnels survive reboots and reconnect automatically.
