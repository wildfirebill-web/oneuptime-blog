# How to Persist ip rule Entries Across Reboots on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ip rule, Policy Routing, Linux, Persistent, systemd-networkd, IPv4, Reboot

Description: Learn how to make ip rule (policy routing) entries persistent across reboots on Linux using systemd-networkd, /etc/network/interfaces, and custom systemd services.

---

`ip rule` entries created with the `ip` command are lost on reboot. Persistence requires configuration in the network manager or startup scripts.

## Viewing Current Rules

```bash
ip rule show
# 0:     from all lookup local

# 32766: from all lookup main
# 32767: from all lookup default
```

## Method 1: systemd-networkd (Recommended)

```ini
# /etc/systemd/network/eth1.network
[Match]
Name=eth1

[Network]
Address=10.0.0.10/24

[RoutingPolicyRule]
From=10.0.0.10/32    # Source IP to match
Table=100            # Routing table to use
Priority=100         # Rule priority (lower = evaluated first)
```

```bash
# Apply
networkctl reload

# Verify
ip rule show
# 100: from 10.0.0.10 lookup 100
```

## Method 2: /etc/network/interfaces (Debian)

```bash
# /etc/network/interfaces
auto eth1
iface eth1 inet static
  address 10.0.0.10
  netmask 255.255.255.0
  up   ip rule add from 10.0.0.10 lookup 100
  up   ip route add default via 10.0.0.1 table 100
  down ip rule del from 10.0.0.10
```

## Method 3: Custom Systemd Service

```bash
# /usr/local/bin/setup-policy-routing.sh
#!/bin/bash
# Add policy routing rules
ip rule add from 10.0.0.10 lookup 100 priority 100
ip rule add from 10.0.0.20 lookup 200 priority 200

# Add routes to custom tables
ip route add default via 10.0.0.1 table 100
ip route add default via 10.0.0.2 table 200
```

```ini
# /etc/systemd/system/policy-routing.service
[Unit]
Description=Policy Routing Rules
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/setup-policy-routing.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
chmod +x /usr/local/bin/setup-policy-routing.sh
systemctl enable --now policy-routing.service
```

## Method 4: NetworkManager (RHEL)

```bash
# nmcli: add routing policy via dispatcher script
# /etc/NetworkManager/dispatcher.d/50-policy-routing
#!/bin/bash
IF=$1
ACTION=$2

if [ "$IF" = "eth1" ] && [ "$ACTION" = "up" ]; then
  ip rule add from 10.0.0.10 lookup 100 priority 100
  ip route add default via 10.0.0.1 table 100
fi

chmod +x /etc/NetworkManager/dispatcher.d/50-policy-routing
```

## Verifying Persistence

```bash
reboot

# After reboot:
ip rule show
# Should show your custom rules

ip route show table 100
# Should show routes in custom table
```

## Key Takeaways

- `ip rule` entries are not persistent by default; use systemd-networkd `[RoutingPolicyRule]` for the cleanest approach.
- In systemd-networkd, `RoutingPolicyRule` entries are automatically managed with the interface lifecycle.
- On Debian, use `up`/`down` hooks in `/etc/network/interfaces` to add/remove rules with the interface.
- A systemd service with `After=network-online.target` is a reliable fallback for any distribution.
