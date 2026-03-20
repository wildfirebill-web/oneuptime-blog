# How to Make Network Bond Configuration Persistent with systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Bonding, systemd-networkd, Linux, Persistent Configuration, Network, IPv4, .netdev

Description: Learn how to configure a persistent Linux network bond using systemd-networkd .netdev and .network files for reproducible, declarative network configuration.

---

systemd-networkd manages bonds through `.netdev` files (create virtual devices) and `.network` files (assign IPs and join bonds). This approach is declarative and persists across reboots without additional packages.

## Creating a Bond with systemd-networkd

### Step 1: Define the Bond Virtual Device

```ini
# /etc/systemd/network/10-bond0.netdev
[NetDev]
Name=bond0
Kind=bond

[Bond]
Mode=active-backup
MIIMonitorSec=100ms
UpDelaySec=200ms
DownDelaySec=200ms
PrimaryReselectPolicy=always
```

### Step 2: Configure Bond IP and DNS

```ini
# /etc/systemd/network/10-bond0.network
[Match]
Name=bond0

[Network]
Address=10.0.0.10/24
Gateway=10.0.0.1
DNS=10.0.0.1
DNS=8.8.8.8

# For DHCP instead:
# DHCP=yes
```

### Step 3: Attach Slave Interfaces

```ini
# /etc/systemd/network/20-eth0.network
[Match]
Name=eth0

[Network]
Bond=bond0
PrimarySlave=yes
```

```ini
# /etc/systemd/network/20-eth1.network
[Match]
Name=eth1

[Network]
Bond=bond0
```

## LACP Bond (802.3ad)

```ini
# /etc/systemd/network/10-bond0-lacp.netdev
[NetDev]
Name=bond0
Kind=bond

[Bond]
Mode=802.3ad
MIIMonitorSec=100ms
LACPTransmitRate=fast
TransmitHashPolicy=layer3+4
```

## Applying Changes

```bash
# Reload systemd-networkd configuration
networkctl reload

# Or restart the service
systemctl restart systemd-networkd

# Verify bond is up
networkctl status bond0
networkctl list
```

## Verifying Persistence

```bash
# Check bond device was created
cat /proc/net/bonding/bond0

# Check IP assignment
ip addr show bond0

# Check slaves
ip link show master bond0

# Verify it survives reboot
systemctl enable systemd-networkd
reboot
ip addr show bond0
```

## Debugging Configuration

```bash
# Check for errors in networkd
journalctl -u systemd-networkd -f

# Validate .network file syntax
networkctl status --all

# Show detailed networkd status
networkctl status bond0
```

## Key Takeaways

- systemd-networkd uses `.netdev` to create the bond device and `.network` to configure IPs and attach slaves.
- Bond options (`Mode`, `MIIMonitorSec`) go in the `[Bond]` section of the `.netdev` file.
- Mark the primary slave with `PrimarySlave=yes` in its `.network` file.
- Run `networkctl reload` to apply changes without restarting networkd; use `networkctl status` to verify.
