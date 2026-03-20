# How to Configure DHCP for Multiple VLANs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, VLAN, Networking, Multi-VLAN, sysadmin

Description: Configuring DHCP for multiple VLANs requires either multiple DHCP servers or relay agents that forward requests to a single centralized server, with one scope per VLAN subnet.

## Architecture Options

| Approach | Pros | Cons |
|----------|------|------|
| One server + relay agents | Centralized management | Relay agent required per VLAN |
| One server per VLAN | No relay needed | Many servers to manage |
| DHCP on router/switch | Built-in | Limited features |

## ISC dhcpd: Multi-VLAN Configuration

Each VLAN subnet needs a `subnet` declaration:

```
# /etc/dhcp/dhcpd.conf

# Global defaults
option domain-name "corp.example.com";
default-lease-time 86400;

# VLAN 10 — Servers
subnet 10.0.10.0 netmask 255.255.255.0 {
    range 10.0.10.50 10.0.10.200;
    option routers 10.0.10.1;
    option domain-name-servers 10.0.0.53;
    default-lease-time 604800;    # Servers: 7-day leases
}

# VLAN 20 — Users
subnet 10.0.20.0 netmask 255.255.255.0 {
    range 10.0.20.50 10.0.20.240;
    option routers 10.0.20.1;
    option domain-name-servers 10.0.0.53;
    default-lease-time 86400;     # Users: 1-day leases
}

# VLAN 30 — VoIP
subnet 10.0.30.0 netmask 255.255.255.0 {
    range 10.0.30.10 10.0.30.250;
    option routers 10.0.30.1;
    option domain-name-servers 10.0.0.53;
    option tftp-server-name "10.0.0.100";
    default-lease-time 3600;      # VoIP: 1-hour leases
}

# VLAN 99 — Guest
subnet 10.0.99.0 netmask 255.255.255.0 {
    range 10.0.99.10 10.0.99.250;
    option routers 10.0.99.1;
    option domain-name-servers 8.8.8.8;
    default-lease-time 1800;      # Guests: 30-min leases
}
```

## Setting Up Relay Agents on a Linux Router

```bash
# eth0 = WAN/uplink, eth0.10/20/30/99 = VLAN sub-interfaces
# DHCP server at 10.0.0.53

# Start dhcrelay for all VLAN interfaces
sudo dhcrelay -i eth0.10 -i eth0.20 -i eth0.30 -i eth0.99 10.0.0.53

# Systemd service with interface list
sudo tee /etc/default/isc-dhcp-relay << 'EOF'
SERVERS="10.0.0.53"
INTERFACES="eth0.10 eth0.20 eth0.30 eth0.99"
OPTIONS=""
EOF
sudo systemctl restart isc-dhcp-relay
```

## Cisco IOS: ip helper-address per VLAN

```
interface Vlan10
  ip address 10.0.10.1 255.255.255.0
  ip helper-address 10.0.0.53

interface Vlan20
  ip address 10.0.20.1 255.255.255.0
  ip helper-address 10.0.0.53

interface Vlan30
  ip address 10.0.30.1 255.255.255.0
  ip helper-address 10.0.0.53
```

## Verifying Multi-VLAN DHCP

```bash
# Bind the DHCP server to all sub-interfaces
sudo tee /etc/default/isc-dhcp-server << 'EOF'
INTERFACESv4="eth0.10 eth0.20 eth0.30 eth0.99"
EOF

sudo systemctl restart isc-dhcp-server

# Test from each VLAN — check which scope is assigned
journalctl -u isc-dhcp-server | grep "DHCPACK"
```

## Key Takeaways

- Create one `subnet` declaration per VLAN in dhcpd.conf.
- Use relay agents (`dhcrelay` or Cisco `ip helper-address`) to forward DHCP broadcasts across VLAN boundaries.
- Bind the DHCP server to the VLAN sub-interfaces, not just the physical interface.
- Different VLANs can have different lease times and options (e.g., short leases for guests).
