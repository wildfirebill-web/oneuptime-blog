# How to Configure ISC DHCP Server (dhcpd) on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, ISC dhcpd, Linux, Sysadmin, Network Configuration

Description: ISC DHCP Server (dhcpd) is the most widely used open-source DHCP server, configurable through a declarative configuration file with subnet declarations, host reservations, and extensive option...

## Installation

```bash
# Debian/Ubuntu

sudo apt install isc-dhcp-server

# RHEL/CentOS/Fedora
sudo dnf install dhcp-server

# Verify installation
dhcpd --version
```

## Complete Configuration Example

```nginx
# /etc/dhcp/dhcpd.conf

# =============================================
# Global Settings
# =============================================
authoritative;               # This server is the authoritative DHCP server
log-facility local7;         # Syslog facility
ping-check true;             # Detect conflicts before offering
ping-timeout 1;

# Default option values (overridable per subnet)
option domain-name "example.local";
option domain-name-servers 10.0.0.53, 10.0.0.54;
option ntp-servers 10.0.0.123;
default-lease-time 86400;    # 24 hours
max-lease-time 604800;       # 7 days

# =============================================
# Subnet Declarations
# =============================================

# Management network
subnet 10.0.0.0 netmask 255.255.255.0 {
    range 10.0.0.100 10.0.0.200;
    option routers 10.0.0.1;
    default-lease-time 604800;   # Infrastructure: 7-day leases
}

# User network
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.50 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.53;
    default-lease-time 86400;
}

# =============================================
# Host Reservations
# =============================================
host web-server {
    hardware ethernet AA:BB:CC:DD:EE:01;
    fixed-address 10.0.0.10;
    option host-name "web-server";
}

host printer-main {
    hardware ethernet AA:BB:CC:DD:EE:02;
    fixed-address 192.168.1.20;
}
```

## Interface Configuration

```bash
# Specify interfaces in /etc/default/isc-dhcp-server
sudo tee /etc/default/isc-dhcp-server << 'EOF'
INTERFACESv4="eth0 eth1"
EOF
```

## Validating and Starting

```bash
# Test configuration syntax
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf && echo "Config OK"

# Start and enable
sudo systemctl enable --now isc-dhcp-server

# Check status
sudo systemctl status isc-dhcp-server

# Follow logs
journalctl -u isc-dhcp-server -f
```

## Managing Leases

```bash
# View active leases
cat /var/lib/dhcp/dhcpd.leases

# Count active leases
grep -c "binding state active" /var/lib/dhcp/dhcpd.leases

# Force expiry of all leases (caution: disrupts all clients)
sudo systemctl stop isc-dhcp-server
sudo sh -c "> /var/lib/dhcp/dhcpd.leases"
sudo systemctl start isc-dhcp-server
```

## Key Takeaways

- `authoritative;` tells dhcpd to send DHCPNAK when clients request addresses outside its subnets.
- Always run `dhcpd -t -cf /etc/dhcp/dhcpd.conf` before restarting to catch syntax errors.
- `log-facility local7` + rsyslog configuration routes logs to a dedicated file.
- Multiple subnet declarations in one config file support multi-VLAN deployments.
