# How to Renew a DHCP Lease on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Linux, Networking, Network Diagnostics, Sysadmin

Description: Renewing a DHCP lease on Linux can be done with dhclient, nmcli, or by restarting the networking service, depending on which network management stack is in use.

## Method 1: dhclient (Direct)

```bash
# Release current lease

sudo dhclient -r eth0

# Request a new lease
sudo dhclient eth0

# Verbose output showing DORA process
sudo dhclient -v eth0

# For a specific interface (replace eth0 with yours)
sudo dhclient -r enp3s0
sudo dhclient enp3s0
```

## Method 2: NetworkManager (nmcli)

Most modern desktop and server Linux distributions use NetworkManager:

```bash
# Find the connection name
nmcli connection show

# Deactivate and reactivate to renew lease
nmcli connection down "Wired connection 1"
nmcli connection up "Wired connection 1"

# Or reload the connection
nmcli device reapply eth0

# Force DHCP renew without disconnecting
nmcli device reapply eth0

# View current lease information
nmcli device show eth0 | grep -i dhcp
```

## Method 3: systemd-networkd

On systems using systemd-networkd:

```bash
# Restart networking to trigger renewal
sudo systemctl restart systemd-networkd

# Or use networkctl
networkctl renew eth0

# View lease info
networkctl status eth0
```

## Method 4: Restart Networking Service

```bash
# Debian/Ubuntu (legacy ifupdown)
sudo ifdown eth0 && sudo ifup eth0

# Modern systems
sudo systemctl restart NetworkManager

# Or use ip commands
sudo ip addr flush dev eth0
sudo dhclient eth0
```

## Viewing Current Lease

```bash
# dhclient lease file
cat /var/lib/dhcp/dhclient.leases

# NetworkManager DHCP info
nmcli -f DHCP4 device show eth0

# systemd-networkd
cat /run/systemd/netif/leases/*
```

## Flushing DNS After Renewal

```bash
# If using systemd-resolved
sudo systemd-resolve --flush-caches

# If using nscd
sudo nscd -i hosts

# With resolvectl (newer systems)
sudo resolvectl flush-caches
```

## Key Takeaways

- `dhclient -r eth0 && dhclient eth0` is the universal method regardless of network manager.
- On NetworkManager systems, `nmcli connection down/up` or `nmcli device reapply` is preferred.
- Use `dhclient -v` to see the full DORA exchange for debugging.
- Check `/var/lib/dhcp/dhclient.leases` to inspect current and past lease information.
