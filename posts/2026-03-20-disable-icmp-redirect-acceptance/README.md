# How to Disable ICMP Redirect Acceptance for Security

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMP, Linux, Security, Sysctl, Redirect, Networking

Description: Disable ICMP redirect message acceptance on Linux to prevent attackers from manipulating your routing table via forged ICMP redirect packets.

ICMP redirect messages tell a host to use a different gateway for a specific destination. While legitimate in some networks, they can be forged by attackers to redirect traffic through a machine they control, enabling man-in-the-middle attacks.

## What Are ICMP Redirects?

```text
Normal operation:
  Router A → sends ICMP redirect to Host → "Use Router B for 10.0.0.0/24"
  Host updates routing cache → traffic flows through Router B

Attack scenario:
  Attacker sends forged ICMP redirect → Host → "Use attacker's machine for 10.0.0.0/24"
  Host sends traffic to attacker → man-in-the-middle
```

## Check Current Redirect Settings

```bash
# Check if ICMP redirects are accepted (1 = yes, 0 = no)

sysctl net.ipv4.conf.all.accept_redirects
sysctl net.ipv4.conf.eth0.accept_redirects

# Check if secure redirects are accepted (from known gateways)
sysctl net.ipv4.conf.all.secure_redirects

# Check if redirects are sent out
sysctl net.ipv4.conf.all.send_redirects
```

## Disable ICMP Redirect Acceptance

```bash
# Disable accepting ICMP redirects on all interfaces
sudo sysctl -w net.ipv4.conf.all.accept_redirects=0
sudo sysctl -w net.ipv4.conf.default.accept_redirects=0

# Disable "secure" redirects too (from listed gateways)
sudo sysctl -w net.ipv4.conf.all.secure_redirects=0
sudo sysctl -w net.ipv4.conf.default.secure_redirects=0

# Also disable per-interface
sudo sysctl -w net.ipv4.conf.eth0.accept_redirects=0
sudo sysctl -w net.ipv4.conf.lo.accept_redirects=0
```

## Disable Sending Redirects

If this Linux host is not a router, it should not send ICMP redirects:

```bash
# Disable sending ICMP redirects
sudo sysctl -w net.ipv4.conf.all.send_redirects=0
sudo sysctl -w net.ipv4.conf.default.send_redirects=0

# Note: if you ARE running a router/gateway, you may want to keep
# send_redirects=1 so clients can use better routes
```

## Make Changes Permanent

```bash
# /etc/sysctl.d/99-disable-redirects.conf

sudo tee /etc/sysctl.d/99-disable-redirects.conf << 'EOF'
# Disable ICMP redirect acceptance
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0

# Disable secure redirects
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

# Disable sending redirects (for non-routers)
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
EOF

# Apply immediately
sudo sysctl -p /etc/sysctl.d/99-disable-redirects.conf
```

## Block with iptables as Defense-in-Depth

Also block ICMP redirect packets at the firewall level:

```bash
# Drop ICMP redirect packets (type 5)
sudo iptables -A INPUT -p icmp --icmp-type redirect -j DROP

# Log and drop for visibility
sudo iptables -A INPUT -p icmp --icmp-type redirect \
  -j LOG --log-prefix "ICMP-REDIRECT-DROP: "
sudo iptables -A INPUT -p icmp --icmp-type redirect -j DROP
```

## Verify the Configuration

```bash
# Confirm all redirect settings are disabled
sysctl -a 2>/dev/null | grep "redirect" | grep "ipv4"

# Expected output (all should be 0):
# net.ipv4.conf.all.accept_redirects = 0
# net.ipv4.conf.all.secure_redirects = 0
# net.ipv4.conf.all.send_redirects = 0
# net.ipv4.conf.default.accept_redirects = 0
# ...

# Check for compliance
grep "accept_redirects" /etc/sysctl.d/*.conf /etc/sysctl.conf
```

Disabling ICMP redirect acceptance is a critical hardening step with no functional impact on normal server operations - Linux hosts don't need to receive routing updates via ICMP redirects.
