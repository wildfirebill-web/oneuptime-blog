# How to Debug systemd-networkd with Journal Logs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: systemd-networkd, Debugging, Journal, Linux, journalctl, Network Troubleshooting

Description: Learn how to debug systemd-networkd configuration issues using journalctl, enabling debug logging, and interpreting common error messages.

---

systemd-networkd logs to the system journal. When network interfaces fail to configure, journal logs reveal configuration errors, DHCP failures, and routing issues.

## Basic Journal Inspection

```bash
# Show all systemd-networkd logs

journalctl -u systemd-networkd

# Follow live (like tail -f)
journalctl -u systemd-networkd -f

# Show last 100 lines
journalctl -u systemd-networkd -n 100

# Show logs since boot
journalctl -u systemd-networkd -b
```

## Enabling Debug Logging

```bash
# Method 1: Environment variable for a single run
SYSTEMD_LOG_LEVEL=debug /usr/lib/systemd/systemd-networkd

# Method 2: Persistent debug level via systemd override
mkdir -p /etc/systemd/system/systemd-networkd.service.d/
cat > /etc/systemd/system/systemd-networkd.service.d/debug.conf << 'EOF'
[Service]
Environment=SYSTEMD_LOG_LEVEL=debug
EOF

systemctl daemon-reload
systemctl restart systemd-networkd

# Disable after debugging:
rm /etc/systemd/system/systemd-networkd.service.d/debug.conf
systemctl daemon-reload
systemctl restart systemd-networkd
```

## Common Error Messages

```bash
# "Failed to open configuration file"
# Cause: Syntax error in .network or .netdev file
# Fix: Check file syntax with networkctl verify

# "Could not find matching network"
# Cause: No .network file matches the interface
# Fix: Check [Match] section - name glob or MAC

# "DHCP timeout"
# Cause: DHCP server unreachable
# Fix: Check physical connectivity, DHCP server logs

# "Failed to set MTU"
# Cause: MTU value too large for hardware
# Fix: Lower MTUBytes value in .link or .network file
```

## Validating Configuration Files

```bash
# Check all .network and .netdev files for syntax errors
networkctl verify

# Output shows warnings and errors:
# /etc/systemd/network/10-eth0.network: OK
# /etc/systemd/network/bad.network: [Match] section is missing

# Check a specific file
networkctl verify /etc/systemd/network/10-eth0.network
```

## Viewing Interface Status

```bash
# Show all interfaces and their networkd status
networkctl list

# Output:
# IDX LINK     TYPE     OPERATIONAL SETUP
#   1 lo       loopback carrier     unmanaged
#   2 eth0     ether    routable    configured
#   3 eth1     ether    degraded    configuring

# Show detailed status for one interface
networkctl status eth0

# Degraded: interface is up but missing some configuration (e.g., no default route)
# Configuring: still waiting for DHCP
# Configured: fully configured
```

## Reloading Without Restart

```bash
# Reload configuration files without dropping connections
networkctl reload

# Reconfigure a specific interface
networkctl reconfigure eth0
```

## Correlating with Kernel Messages

```bash
# View kernel network events alongside networkd logs
journalctl -k -u systemd-networkd -b | grep -E "eth0|bond|vxlan"

# Check for kernel errors
dmesg | grep -E "eth0|nf_|bond" | tail -20
```

## Key Takeaways

- `journalctl -u systemd-networkd -f` provides real-time network configuration log monitoring.
- Enable debug logging with `SYSTEMD_LOG_LEVEL=debug` in a systemd override file to see detailed DHCP and routing events.
- `networkctl verify` validates all `.network` and `.netdev` files for syntax errors before applying them.
- `networkctl list` shows operational status; `degraded` means the interface is up but not fully configured.
