# How to Harden IPv4 Network Security with sysctl Parameters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Sysctl, Linux, Security, IPv4, Hardening, Kernel

Description: Apply a comprehensive set of sysctl parameters to harden the Linux kernel's IPv4 networking stack against spoofing, floods, and reconnaissance attacks.

The Linux kernel exposes hundreds of tunable security parameters via sysctl. Applying the right settings can block entire attack categories before packets reach iptables or applications.

## Key Security Parameters

The most important parameters fall into these categories:

```text
Category              Parameters
--------------------  ------------------------------------------
Spoofing prevention   rp_filter, martian logging
SYN flood defense     syncookies, syn backlog, retries
ICMP security         ignore broadcasts, echo ignore, redirects
Routing security      accept_redirects, send_redirects
Logging               log_martians
```

## Anti-Spoofing and Bogon Filtering

```bash
# /etc/sysctl.d/99-security.conf

# Enable strict reverse path filtering (drop spoofed source IPs)

net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Log packets with impossible source addresses (martians)
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
```

## SYN Flood Protection

```bash
# Enable SYN cookies (stateless SYN flood defense)
net.ipv4.tcp_syncookies = 1

# Increase SYN backlog queue
net.ipv4.tcp_max_syn_backlog = 4096

# Reduce SYN-ACK retries (faster cleanup of half-open connections)
net.ipv4.tcp_synack_retries = 2

# Reduce SYN retransmit attempts
net.ipv4.tcp_syn_retries = 5
```

## ICMP Hardening

```bash
# Ignore ICMP broadcast requests (prevents Smurf attack)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore ICMP bogus error responses
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Optional: ignore all pings (set to 1 to disable ping)
net.ipv4.icmp_echo_ignore_all = 0
```

## Disable ICMP Redirects

ICMP redirects can be used to manipulate routing tables:

```bash
# Do not accept ICMP redirect messages
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0

# Do not send ICMP redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Do not accept secure redirects (from gateways)
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
```

## Disable Source Routing

Source routing allows the sender to specify the packet's path:

```bash
# Disable IPv4 source routing (used in spoofing/routing attacks)
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
```

## TCP Connection Security

```bash
# Disable TCP timestamps (hides kernel uptime from scanners)
net.ipv4.tcp_timestamps = 0

# Enable TCP RFC 1337 (TIME_WAIT assassination fix)
net.ipv4.tcp_rfc1337 = 1

# Protect against TCP RST attacks
net.ipv4.tcp_challenge_ack_limit = 1000
```

## Apply All Settings

Create a comprehensive sysctl configuration file:

```bash
# Create security hardening config
sudo tee /etc/sysctl.d/99-ipv4-security.conf << 'EOF'
# Anti-spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.log_martians = 1

# SYN flood protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_synack_retries = 2

# ICMP hardening
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Disable redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.secure_redirects = 0

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0
EOF

# Apply all settings
sudo sysctl -p /etc/sysctl.d/99-ipv4-security.conf

# Verify a key setting
sysctl net.ipv4.tcp_syncookies
# net.ipv4.tcp_syncookies = 1
```

These sysctl settings are the foundation of Linux network hardening and should be applied to every server exposed to untrusted networks.
