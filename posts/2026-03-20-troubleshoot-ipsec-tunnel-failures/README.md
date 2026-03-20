# How to Troubleshoot IPsec VPN Tunnel Establishment Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPsec, VPN, Troubleshooting, strongSwan, IKEv2, Linux

Description: Diagnose and resolve IPsec VPN tunnel establishment failures including IKE negotiation errors, authentication failures, and routing issues.

IPsec tunnel failures fall into a few categories: connectivity failures (can't reach the peer), negotiation failures (incompatible parameters), and authentication failures (wrong keys or certificates).

## Step 1: Check Basic Connectivity

```bash
# Can the two gateways communicate?

ping <REMOTE_GATEWAY_IP>

# Verify UDP port 500 (IKE) is accessible
nc -zuv <REMOTE_GATEWAY_IP> 500
# "Connection to ... succeeded" = port open
# "Connection refused" or timeout = firewall blocking

# Check for NAT between gateways (if NAT-T is needed)
nc -zuv <REMOTE_GATEWAY_IP> 4500
```

## Step 2: Enable Detailed strongSwan Logging

```bash
# Set verbose logging
sudo ipsec stroke loglevel ike 4
sudo ipsec stroke loglevel knl 2
sudo ipsec stroke loglevel cfg 2

# Watch logs as you bring up the tunnel
sudo journalctl -u strongswan -f &
sudo ipsec up site-to-site

# Key messages to look for:
# "initiating IKE_SA" → tunnel is being initiated
# "received IKE_SA_INIT response" → peer replied
# "IKE_AUTH request sent" → authentication in progress
# "IKE_SA established" → Phase 1 success
# "CHILD_SA established" → Phase 2 success, tunnel active
```

## Step 3: Diagnose Common Errors

### "No proposal chosen"

```bash
# Error: received NO_PROPOSAL_CHOSEN notify
# Cause: Algorithm mismatch between peers

# Check what you're proposing vs what the peer accepts
sudo journalctl -u strongswan | grep -i "proposal\|NO_PROPOSAL"

# Fix: Change ike= and esp= to match the peer's capabilities
# Example: Try removing the '!' to allow algorithm negotiation
# ike=aes256-sha256-modp2048   (without ! = allow fallback)
```

### "Authentication Failed" or PSK Mismatch

```bash
# Error: authentication of ... failed
# Cause: Wrong PSK, wrong leftid/rightid, or certificate issue

# For PSK: verify exact match including whitespace
grep PSK /etc/ipsec.secrets

# Check leftid/rightid match on both sides
# Gateway A: leftid=@gateway-a, rightid=@gateway-b
# Gateway B: leftid=@gateway-b, rightid=@gateway-a
# These MUST mirror each other
```

### Tunnel Connects but No Traffic Flows

```bash
# Tunnel is ESTABLISHED but traffic doesn't flow
# Check XFRM policies
sudo ip xfrm policy

# Verify left/rightsubnet are correct
sudo ipsec statusall | grep "Traffic Selectors"

# Check kernel forwarding
sysctl net.ipv4.ip_forward

# Test connectivity directly
ping -I <LAN_INTERFACE> <REMOTE_SUBNET_HOST>
```

### NAT Issues

```bash
# If there's NAT between gateways, IKE moves from UDP 500 to UDP 4500
# Verify NAT-T is working
sudo ipsec statusall | grep "NAT-T"
# Should show: local 4500, remote 4500 (ports changed from 500)

# If NAT-T isn't detected, force it
# In ipsec.conf:
# forceencaps=yes   # Force NAT-T even when no NAT detected
```

## Step 4: Check Firewall Rules

```bash
# On BOTH gateways, ensure these are allowed:
sudo iptables -L INPUT -n | grep -E "500|4500|esp"

# Required rules:
sudo iptables -A INPUT -p udp --dport 500 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 4500 -j ACCEPT
sudo iptables -A INPUT -p esp -j ACCEPT
sudo iptables -A INPUT -p ah -j ACCEPT
sudo iptables -A FORWARD -m policy --dir in --pol ipsec -j ACCEPT
sudo iptables -A FORWARD -m policy --dir out --pol ipsec -j ACCEPT
```

## Step 5: Capture IKE Traffic

```bash
# Capture IKE and ESP traffic for analysis
sudo tcpdump -i eth0 -n '(udp port 500 or udp port 4500 or proto 50)' -w ipsec.pcap

# Analyze with Wireshark - IKEv2 dissector shows the full negotiation
wireshark ipsec.pcap
```

## Quick Diagnostic Checklist

1. Ping the remote gateway? → If no, fix routing/firewall first
2. `ipsec statusall` shows "ESTABLISHED"? → If no, check algorithm/auth
3. XFRM policies installed? → `ip xfrm policy` shows entries
4. IP forwarding enabled? → `sysctl net.ipv4.ip_forward` = 1
5. No NAT between gateways for ESP? → Use NAT-T or forceencaps

Step-by-step log analysis with verbose logging resolves virtually all IPsec tunnel establishment failures.
