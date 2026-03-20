# How to Configure Multiple IPv4 Addresses with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: systemd-networkd, IPv4, Multiple Addresses, Alias, Linux, .network, Networking

Description: Learn how to assign multiple IPv4 addresses to a single network interface using systemd-networkd .network files, including virtual hosting configurations and secondary addresses.

---

Multiple IPv4 addresses on one interface are common for web servers hosting multiple sites, IP aliasing, and high-availability configurations.

## Assigning Multiple Addresses

```ini
# /etc/systemd/network/10-eth0.network

[Match]
Name=eth0

[Network]
# Multiple Address lines for multiple IPs
Address=10.0.0.5/24
Address=10.0.0.6/24
Address=10.0.0.7/24

Gateway=10.0.0.1
DNS=10.0.0.1
```

## Applying and Verifying

```bash
networkctl reload

# Verify all addresses are assigned
ip addr show eth0
# inet 10.0.0.5/24 ...
# inet 10.0.0.6/24 ...
# inet 10.0.0.7/24 ...
```

## Different Subnets on One Interface

```ini
[Match]
Name=eth0

[Network]
Address=10.0.0.5/24     # Primary address
Address=192.168.1.5/24  # Secondary, different subnet
Address=172.16.0.5/16   # Tertiary
Gateway=10.0.0.1
```

## Address with Peer (Point-to-Point)

```ini
[Address]
Address=10.0.0.5/30
Peer=10.0.0.6/30   # The remote end's IP (for p2p links)
```

## Address with Broadcast

```ini
[Address]
Address=10.0.0.5/24
Broadcast=10.0.0.255   # Explicit broadcast (usually inferred)
Label=eth0:1           # Label for ip addr show (legacy compat)
```

## Scope and Flags

```ini
[Address]
Address=169.254.0.1/16
Scope=link              # Link-local scope
```

## Address Block with DHCP

```ini
[Network]
DHCP=yes                # Get primary IP via DHCP
Address=10.0.0.100/24  # Always assign this additional static IP
```

## Managing Individual Addresses

```bash
# Add address manually (temporary)
ip addr add 10.0.0.8/24 dev eth0

# Remove specific address
ip addr del 10.0.0.8/24 dev eth0

# Show primary address only
ip addr show eth0 primary

# Show all addresses
ip addr show eth0
```

## Source Address Selection for Outgoing Traffic

```bash
# Linux uses the primary address for outgoing connections by default
# To use a specific source address:
curl --interface 10.0.0.6 http://example.com

# Or via ip rule:
ip rule add from 10.0.0.6 lookup 200
ip route add default via 10.0.0.1 table 200
```

## Key Takeaways

- Add multiple `Address=` lines in the `[Network]` section of a `.network` file to assign multiple IPs.
- The first `Address=` and `Gateway=` in the file is considered the primary; subsequent addresses are secondary.
- Use `[Address]` sections for fine-grained control over broadcast, scope, and peer for each address.
- Run `networkctl reload` to apply; use `ip addr show eth0` to verify all addresses are assigned.
