# How to Configure DHCP Client Options with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, systemd-networkd, Linux, DHCP Client, IPv4, .network, Client Options

Description: Learn how to configure DHCP client behavior in systemd-networkd, including requesting specific options, setting client IDs, controlling route acceptance, and customizing lease behavior.

---

systemd-networkd includes a built-in DHCP client. Client behavior is controlled through the `[DHCP]` section in `.network` files.

## Basic DHCP Configuration

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
DHCP=yes           # Enable DHCP for IPv4 (and IPv6)
# Or:
# DHCP=ipv4        # IPv4 only
# DHCP=ipv6        # IPv6 only
# DHCP=no          # Disable DHCP
```

## Customizing DHCP Client Behavior

```ini
[Match]
Name=eth0

[Network]
DHCP=yes

[DHCP]
# Client identifier: mac (default) or duid
ClientIdentifier=mac

# Hostname to send to DHCP server
Hostname=myserver

# Request specific DHCP options (option numbers)
RequestOptions=28 42 121    # Broadcast, NTP, Classless Static Route

# Do not accept DNS from DHCP (use static DNS instead)
UseDNS=no

# Do not accept routes from DHCP
UseRoutes=no

# Do not accept NTP servers from DHCP
UseNTP=no

# Do not set hostname from DHCP
UseHostname=no

# Do not accept default gateway from DHCP
UseGateway=no
```

## Combining DHCP with Static Settings

```ini
[Match]
Name=eth0

[Network]
DHCP=yes
Address=10.0.0.100/24   # Always assign this static IP in addition to DHCP

[DHCP]
UseDNS=no               # Ignore DHCP DNS

[DNS]
DNS=8.8.8.8             # Use static DNS
```

## DHCP with Static Routes Override

```ini
[Network]
DHCP=yes

[DHCP]
UseRoutes=no    # Ignore routes from DHCP

[Route]
Destination=0.0.0.0/0
Gateway=192.168.1.254   # Use this static default route instead
```

## DHCP Client ID (DUID)

```ini
[DHCP]
# Use DUID (DHCPv6-style identifier for DHCPv4)
ClientIdentifier=duid

# Or specify raw DHCP option 60 vendor class
VendorClassIdentifier=MyApp/1.0
```

## DHCP Anonymization

```ini
[DHCP]
# Randomize client ID to avoid tracking (privacy)
Anonymize=yes
```

## Rapid Commit (Option 80)

```ini
[DHCP]
# Enable DHCP rapid commit (2-way instead of 4-way handshake)
RapidCommit=yes
```

## Debugging DHCP

```bash
# Watch DHCP events
journalctl -u systemd-networkd -f | grep -i dhcp

# Check acquired lease
networkctl status eth0
# Shows: DHCP4 Client...
#        Lease Server Address: 192.168.1.1

# Renew DHCP lease
networkctl renew eth0
```

## Key Takeaways

- The `[DHCP]` section in `.network` files controls all DHCP client behavior in systemd-networkd.
- Use `UseDNS=no`, `UseRoutes=no`, `UseGateway=no` to selectively ignore DHCP-provided values.
- `Hostname=` sends a specific hostname to the DHCP server for DNS registration.
- Use `networkctl renew eth0` to manually trigger a DHCP lease renewal without restarting networkd.
