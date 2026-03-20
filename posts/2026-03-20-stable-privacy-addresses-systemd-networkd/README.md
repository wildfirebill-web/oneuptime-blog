# How to Configure Stable Privacy Addresses on systemd-networkd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, systemd-networkd, Privacy, RFC7217, Linux, Networking

Description: Configure RFC 7217 stable privacy IPv6 addresses using systemd-networkd to prevent cross-network device tracking on Linux servers and desktops.

## Introduction

systemd-networkd is the network management daemon included with systemd, widely used on servers and minimal Linux installations. It has native support for IPv6 stable privacy addresses via the `IPv6PrivacyExtensions` directive, giving administrators fine-grained control over address generation behavior.

## Understanding the IPv6PrivacyExtensions Directive

systemd-networkd's `[Network]` section supports these values for `IPv6PrivacyExtensions`:

| Value | Behavior |
|---|---|
| `no` | Use EUI-64 (MAC-based) addresses |
| `prefer-public` | Generate temporary addresses but prefer the stable one |
| `yes` | Generate temporary addresses and prefer them |
| `kernel` | Defer to kernel sysctl settings |

For RFC 7217-style stable privacy, set `IPv6PrivacyExtensions=no` combined with kernel-level stable-privacy `addr_gen_mode`.

## Configuring a Network File

Create or edit a `.network` file for your interface in `/etc/systemd/network/`:

```ini
# /etc/systemd/network/10-eth0.network
# Configure eth0 with stable privacy IPv6 address generation

[Match]
Name=eth0

[Network]
DHCP=ipv4
IPv6AcceptRA=yes

# Use stable-privacy address generation (RFC 7217)
# This disables temporary addresses in favor of a single stable opaque address
IPv6PrivacyExtensions=no
```

Then set the kernel addr_gen_mode to stable-privacy:

```bash
# Set addr_gen_mode to 2 (stable-privacy) for eth0
echo 2 | sudo tee /proc/sys/net/ipv6/conf/eth0/addr_gen_mode
```

## Using Kernel-Mode Privacy for Full RFC 7217 Support

The cleanest approach combines systemd-networkd's `kernel` setting with sysctl:

```ini
# /etc/systemd/network/10-eth0.network

[Match]
Name=eth0

[Network]
DHCP=ipv4
IPv6AcceptRA=yes
IPv6PrivacyExtensions=kernel
```

Configure sysctl to use stable-privacy mode globally:

```bash
# /etc/sysctl.d/60-ipv6-privacy.conf
# 2 = stable-privacy (RFC 7217) for all interfaces
net.ipv6.conf.default.addr_gen_mode = 2
net.ipv6.conf.all.addr_gen_mode = 2
```

Apply sysctl settings:

```bash
sudo sysctl --system
```

## Restarting systemd-networkd

After making changes to `.network` files, restart the daemon:

```bash
# Restart systemd-networkd to apply network file changes
sudo systemctl restart systemd-networkd

# Verify the daemon is healthy
sudo systemctl status systemd-networkd
```

## Verifying the Stable Address

Check that the IPv6 address is opaque (not EUI-64 based) and stable:

```bash
# Display IPv6 addresses for eth0
ip -6 addr show eth0

# Note down the IID (last 64 bits of the address)
# Reboot and verify the same IID appears on the same network
```

For a quick comparison, compute what the EUI-64 address would look like:

```bash
# Get the MAC address of eth0
MAC=$(cat /sys/class/net/eth0/address)
echo "MAC: $MAC"
# If your IPv6 IID differs from the EUI-64 derived from this MAC,
# stable-privacy is working correctly
```

## Configuring Multiple Interfaces

Apply the settings to all interfaces using a wildcard match:

```ini
# /etc/systemd/network/10-all-eth.network

[Match]
Name=en*

[Network]
DHCP=ipv4
IPv6AcceptRA=yes
IPv6PrivacyExtensions=kernel
```

## Viewing networkctl Status

Use `networkctl` to confirm the configuration is applied:

```bash
# Show detailed status for eth0
networkctl status eth0

# Look for "IPv6PrivacyExtensions" in the output
```

## Conclusion

systemd-networkd provides straightforward support for RFC 7217 stable privacy addresses through its `.network` file directives combined with kernel sysctl settings. This approach is ideal for servers and headless Linux systems managed without NetworkManager. The resulting addresses are opaque, stable per network, and do not expose the hardware MAC address.
