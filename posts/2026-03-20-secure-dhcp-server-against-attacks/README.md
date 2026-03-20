# How to Secure Your DHCP Server Against Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Security, Networking, Sysadmin, Network Security

Description: DHCP servers are vulnerable to starvation attacks, rogue server impersonation, and unauthorized access, and can be hardened through DHCP snooping, rate limiting, MAC filtering, and firewall rules.

## Common DHCP Attack Vectors

| Attack | Description | Impact |
|--------|-------------|--------|
| DHCP Starvation | Exhaust the pool with fake MACs | Legitimate clients can't get IPs |
| Rogue DHCP Server | Attacker serves malicious gateway/DNS | MitM traffic redirect |
| DHCP Spoofing | Fake DHCP replies | Incorrect configuration |
| Unauthorized Access | Direct access to DHCP admin | Config tampering |

## Defense 1: Restrict DHCP Server to Specific Interfaces

```bash
# Only listen on internal interface - never on WAN

sudo tee /etc/default/isc-dhcp-server << 'EOF'
INTERFACESv4="eth1"   # LAN only, NOT eth0 (WAN)
EOF
```

## Defense 2: Firewall the DHCP Server

```bash
# Allow DHCP only from trusted subnets
iptables -A INPUT -p udp --dport 67 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p udp --dport 67 -j DROP    # Deny all other sources

# Block external DHCP traffic
iptables -A INPUT -i eth0 -p udp --dport 67 -j DROP    # Drop on WAN
iptables -A INPUT -i eth0 -p udp --dport 68 -j DROP
```

## Defense 3: MAC Address Filtering

```bash
# Allow only known MACs in dhcpd.conf
# Deny unknown hosts by omitting a range and requiring explicit reservations

# dhcpd.conf: deny all clients without a host declaration
subnet 10.0.10.0 netmask 255.255.255.0 {
    deny unknown-clients;
    option routers 10.0.10.1;
    # Only these known hosts will get IPs:
}

host known-workstation-1 {
    hardware ethernet aa:bb:cc:dd:ee:01;
    fixed-address 10.0.10.10;
}
```

## Defense 4: Rate Limiting DHCP Requests

```bash
# iptables: limit DHCP discover rate per source MAC (not directly possible)
# Use switch-level rate limiting (see DHCP snooping post)

# Or limit by source IP rate (useful for relay environments)
iptables -A INPUT -p udp --dport 67 -m recent --set --name DHCP
iptables -A INPUT -p udp --dport 67 -m recent --update --seconds 10 --hitcount 20 --name DHCP -j DROP
```

## Defense 5: Secure the Admin Interface

```bash
# Restrict OMAPI (remote DHCP management) access
# In dhcpd.conf:
omapi-port 7911;
omapi-key dhcpKey;

# Key generation
tsig-keygen -a HMAC-SHA256 dhcpKey >> /etc/dhcp/dhcpd.conf
```

## Defense 6: Enable Conflict Detection

```text
# /etc/dhcp/dhcpd.conf
# Ping before offering to prevent duplicate assignments
ping-check true;
ping-timeout 1;
```

## Defense 7: Monitor for Rogue DHCP Servers

```bash
# Scan for DHCP servers on the network
sudo nmap --script broadcast-dhcp-discover -e eth0

# Or use dhcp-probe (detects multiple DHCP servers)
sudo apt install dhcp-probe
dhcp_probe eth0
```

## Key Takeaways

- Bind the DHCP server only to internal interfaces.
- Use `deny unknown-clients` and MAC reservations to prevent starvation attacks.
- Enable DHCP snooping on switches to block rogue DHCP servers.
- Monitor for unauthorized DHCP servers with `nmap --script broadcast-dhcp-discover`.
