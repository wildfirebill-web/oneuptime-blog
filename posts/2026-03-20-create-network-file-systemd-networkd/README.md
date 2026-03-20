# How to Create .network Files in systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Linux, systemd-networkd, Network Configuration, Static IP, DHCP

Description: Learn how to write systemd-networkd .network files to configure static IPs, DHCP, routes, and DNS for network interfaces.

---

.network files are the primary configuration unit in systemd-networkd. They match network interfaces by name or MAC address and define IP addresses, routes, DNS, and link settings.

---

## File Location and Structure

```bash
# Files live in:

/etc/systemd/network/

# Lower numbers are processed first
10-eth0.network
20-eth1.network
```

---

## DHCP Configuration

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
DHCP=yes
```

---

## Static IP Configuration

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Address=192.168.1.10/24
Gateway=192.168.1.1
DNS=8.8.8.8
DNS=1.1.1.1
```

---

## Multiple Addresses

```ini
[Match]
Name=eth0

[Network]
Address=192.168.1.10/24
Address=192.168.1.11/24
Gateway=192.168.1.1
```

---

## Custom Routes

```ini
[Match]
Name=eth0

[Network]
Address=10.0.0.5/24
Gateway=10.0.0.1

[Route]
Destination=172.16.0.0/12
Gateway=10.0.0.254
Metric=100
```

---

## Match by MAC Address

```ini
[Match]
MACAddress=aa:bb:cc:dd:ee:ff

[Network]
DHCP=yes
```

---

## Attach to a Bond or Bridge

```ini
[Match]
Name=eth0

[Network]
Bond=bond0
```

---

## Apply and Verify

```bash
sudo systemctl restart systemd-networkd

networkctl list
networkctl status eth0
ip addr show eth0
ip route show
```

---

## Summary

`.network` files match interfaces with `[Match]` directives (by name, MAC, or driver) and configure them with `[Network]` (addresses, DNS, DHCP), `[Route]` (custom routes), and `[Link]` (MTU, checksums) sections. Use `networkctl status <interface>` to verify the applied configuration and `systemctl restart systemd-networkd` to apply changes.
