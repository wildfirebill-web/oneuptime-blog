# How to Disable IPv6 on Linux with sysctl

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Linux, Sysctl, Network Configuration, Security

Description: A step-by-step guide to disabling IPv6 on Linux systems using sysctl parameters, both temporarily at runtime and permanently across reboots.

## When to Disable IPv6

Disabling IPv6 is sometimes necessary for:
- Legacy applications incompatible with dual-stack
- Security hardening when IPv6 is not used
- Troubleshooting suspected IPv6-related issues
- Compliance requirements in specific environments

**Note**: Disabling IPv6 can break applications that depend on it (including some system services that use `::1` for loopback). Test thoroughly before disabling in production.

## Temporary Disable (Until Reboot)

```bash
# Disable IPv6 on all interfaces immediately

sysctl -w net.ipv6.conf.all.disable_ipv6=1

# Disable on the default profile for new interfaces
sysctl -w net.ipv6.conf.default.disable_ipv6=1

# Disable on a specific interface (e.g., eth0)
sysctl -w net.ipv6.conf.eth0.disable_ipv6=1

# Also disable on loopback (may affect some services)
sysctl -w net.ipv6.conf.lo.disable_ipv6=1

# Verify IPv6 addresses are removed
ip -6 addr show
# Should return empty or only show ::1 if lo isn't disabled
```

## Permanent Disable via /etc/sysctl.conf

```bash
# Add to /etc/sysctl.conf for persistence
cat >> /etc/sysctl.conf << 'EOF'

# Disable IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF

# Apply immediately
sysctl -p /etc/sysctl.conf

# Verify
sysctl net.ipv6.conf.all.disable_ipv6
# Should return: 1
```

## Permanent Disable via /etc/sysctl.d/

The preferred modern approach is to use a dedicated file in `/etc/sysctl.d/`:

```bash
# Create a dedicated sysctl config file
cat > /etc/sysctl.d/99-disable-ipv6.conf << 'EOF'
# Disable IPv6 system-wide
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF

# Apply immediately without reboot
sysctl --system
# or
sysctl -p /etc/sysctl.d/99-disable-ipv6.conf

# Verify
ip -6 addr show
```

## Disabling IPv6 on Ubuntu

Ubuntu uses netplan for network management. To disable IPv6 there:

```yaml
# /etc/netplan/00-installer-config.yaml

network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: false      # Disable DHCPv6
      link-local: []    # Remove link-local IPv6 generation
      accept-ra: false  # Don't accept Router Advertisements
```

```bash
netplan apply
```

## Disabling IPv6 on RHEL/CentOS/Fedora

For RHEL-based systems, also update the NIC configuration:

```bash
# Method 1: sysctl (universal)
echo 'net.ipv6.conf.all.disable_ipv6 = 1' > /etc/sysctl.d/99-disable-ipv6.conf
sysctl --system

# Method 2: NetworkManager
nmcli connection modify eth0 ipv6.method "disabled"
nmcli connection up eth0
```

## Verifying IPv6 Is Disabled

```bash
# Check sysctl parameter
sysctl net.ipv6.conf.all.disable_ipv6
# Should be: 1

# Check no IPv6 addresses are configured
ip -6 addr show
# Should be empty (except possibly ::1 if loopback not disabled)

# Check no IPv6 ports are listening
ss -6 -tlnp
# Should be empty

# Test that IPv6 DNS resolution doesn't work
dig AAAA google.com @::1 2>&1 | grep -E 'error|refused|failed'
```

## Re-enabling IPv6

If you need to re-enable IPv6 later:

```bash
# Remove the sysctl.d file
rm /etc/sysctl.d/99-disable-ipv6.conf

# Re-enable immediately
sysctl -w net.ipv6.conf.all.disable_ipv6=0
sysctl -w net.ipv6.conf.default.disable_ipv6=0

# Apply
sysctl --system
```

## Summary

Disable IPv6 on Linux with `sysctl -w net.ipv6.conf.all.disable_ipv6=1` for immediate effect, and create `/etc/sysctl.d/99-disable-ipv6.conf` for persistence across reboots. Apply with `sysctl --system`. Verify with `ip -6 addr show` (should be empty). Note that disabling IPv6 may affect services using `::1` for loopback communication, so test your applications after disabling.
