# How to Use ip_conntrack_ftp for Passive FTP Through iptables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: iptables, FTP, Passive Mode, Nf_conntrack_ftp, Connection Tracking, IPv4

Description: Configure the nf_conntrack_ftp kernel module to automatically track FTP passive data connections through iptables, eliminating the need to open a wide port range.

## Introduction

Without `nf_conntrack_ftp`, iptables has no way to know which ephemeral ports an FTP server will use for passive data connections. The connection tracking helper inspects FTP control traffic and automatically creates RELATED entries for the data connections, so they are accepted without explicit rules for each port.

## How Connection Tracking Works for FTP

```text
1. Client connects to Server:21  →  iptables sees NEW connection
2. Server sends: 227 Entering Passive Mode (...,117,49)
   → nf_conntrack_ftp parses this response
   → Creates an EXPECTED entry for Server:30001
3. Client connects to Server:30001  →  iptables sees RELATED connection
   → Automatically accepted by: -m state --state RELATED -j ACCEPT
```

## Loading the Module

```bash
# Load nf_conntrack_ftp helper

sudo modprobe nf_conntrack_ftp

# For older kernels (the module was renamed)
sudo modprobe ip_conntrack_ftp   # legacy name

# Verify it's active
lsmod | grep conntrack_ftp
# Expected output:
# nf_conntrack_ftp       20480  0
# nf_conntrack           163840  3 nf_conntrack_ftp,...

# Check for expected FTP connections
sudo cat /proc/net/nf_conntrack | grep ftp
```

## Making the Module Persistent

```bash
# Debian/Ubuntu
echo "nf_conntrack_ftp" | sudo tee -a /etc/modules

# RHEL/CentOS 7
echo "nf_conntrack_ftp" | sudo tee /etc/modules-load.d/ftp.conf

# systemd-based alternative
cat > /etc/modules-load.d/nf_conntrack_ftp.conf << 'EOF'
nf_conntrack_ftp
EOF

# Verify at boot (check after reboot)
sudo systemctl status systemd-modules-load
```

## iptables Rules with Connection Tracking

```bash
#!/bin/bash
# Minimal FTP firewall using connection tracking

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow established and related connections (handles FTP data!)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow FTP control port
iptables -A INPUT -p tcp --dport 21 -m state --state NEW -j ACCEPT

# No explicit rule needed for passive ports - nf_conntrack_ftp handles them!
```

## Verifying Connection Tracking

```bash
# While an FTP client is connected in passive mode:
sudo watch -n1 "cat /proc/net/nf_conntrack | grep ftp"

# You should see entries like:
# ipv4 tcp src=client_ip dst=server_ip sport=... dport=21 [ESTABLISHED]
# ipv4 tcp src=client_ip dst=server_ip sport=... dport=30001 [EXPECTED]

# Count active FTP connections tracked:
sudo cat /proc/net/nf_conntrack | grep -c "dport=21"
```

## Non-Standard FTP Port

```bash
# If your FTP server uses a non-standard port, tell the module:
# Unload and reload with custom port
sudo modprobe -r nf_conntrack_ftp
sudo modprobe nf_conntrack_ftp ports=2121

# Or set a kernel parameter:
echo "options nf_conntrack_ftp ports=2121" | \
  sudo tee /etc/modprobe.d/nf_conntrack_ftp.conf

# Then allow port 2121 in iptables:
iptables -A INPUT -p tcp --dport 2121 -m state --state NEW -j ACCEPT
```

## Troubleshooting

```bash
# Issue: Passive mode still fails even with module loaded
# Check if conntrack is tracking:
sudo cat /proc/net/nf_conntrack | grep ftp

# Issue: RELATED connections not working
# Verify the module is loaded AND rules include RELATED state:
sudo iptables -L INPUT -n | grep RELATED

# Issue: Module loads but PASV still blocked
# Check if nf_conntrack_helper is enabled (disabled by default in newer kernels)
sudo sysctl net.netfilter.nf_conntrack_helper
# If 0, enable:
sudo sysctl -w net.netfilter.nf_conntrack_helper=1
echo "net.netfilter.nf_conntrack_helper=1" | sudo tee -a /etc/sysctl.conf

# Alternative: Use CT target to explicitly enable helper for port 21:
sudo iptables -t raw -A PREROUTING -p tcp --dport 21 \
  -j CT --helper ftp
```

## Conclusion

`nf_conntrack_ftp` eliminates the need for wide passive port range rules. Load it with `modprobe nf_conntrack_ftp`, persist it in `/etc/modules`, and ensure iptables has `-m state --state ESTABLISHED,RELATED -j ACCEPT`. On newer kernels with `nf_conntrack_helper=0`, use the CT target to explicitly attach the FTP helper to port 21 traffic.
