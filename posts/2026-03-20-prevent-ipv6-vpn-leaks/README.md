# How to Prevent IPv6 VPN Leaks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, VPN, VPN Leaks, Privacy, Security, Network Configuration

Description: A guide to preventing IPv6 traffic from bypassing VPN tunnels, ensuring all IPv6 traffic is either tunneled or blocked to prevent identity exposure.

IPv6 VPN leaks occur when a device with both IPv4 and IPv6 connectivity connects to a VPN that only tunnels IPv4 traffic. IPv6 traffic continues flowing directly to its destination, bypassing the VPN entirely. This reveals the user's real IPv6 address to every site they visit, undermining the privacy purpose of the VPN.

## Understanding IPv6 VPN Leaks

```
Without leak prevention:
  IPv4 traffic → VPN tunnel → Internet (exits at VPN server)
  IPv6 traffic → Direct to Internet (leaks real IPv6 address!)

With leak prevention:
  IPv4 traffic → VPN tunnel → Internet (exits at VPN server)
  IPv6 traffic → Blocked OR tunneled through VPN
```

## Method 1: Block All IPv6 at Firewall Level

The simplest approach: if your VPN doesn't support IPv6, block all IPv6 outbound:

```bash
# Linux: block all outbound IPv6 except through VPN interface
sudo ip6tables -A OUTPUT ! -o tun0 -j DROP

# Allow loopback
sudo ip6tables -A OUTPUT -o lo -j ACCEPT

# Allow established traffic
sudo ip6tables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Block everything else not going through VPN
sudo ip6tables -A OUTPUT ! -o tun0 -j DROP
```

## Method 2: Disable IPv6 on Network Interfaces

```bash
# Disable IPv6 on all interfaces (nuclear option)
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1

# Or disable only on specific interface
sudo sysctl -w net.ipv6.conf.eth0.disable_ipv6=1

# Make persistent
cat >> /etc/sysctl.conf << 'EOF'
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
EOF
sudo sysctl -p
```

## Method 3: Null Route IPv6

Route all IPv6 to a blackhole:

```bash
# Route all IPv6 to null (blackhole)
sudo ip -6 route add blackhole ::/0 metric 1

# This sends all IPv6 traffic to /dev/null instead of leaking
# VPN-specific IPv6 routes (if VPN supports IPv6) will have lower metric
```

## Method 4: VPN with Dual-Stack Support

The best solution: use a VPN that tunnels IPv6 traffic:

### OpenVPN

```ini
# Add to OpenVPN client config:
# Route all IPv6 through VPN
route-ipv6 ::/0

# Push from server side:
push "route-ipv6 ::/0"
```

### WireGuard

```ini
[Peer]
# Include IPv6 in AllowedIPs
AllowedIPs = 0.0.0.0/0, ::/0
```

## Method 5: Kill Switch with IPv6

A kill switch blocks all non-VPN traffic if the VPN disconnects:

```bash
# /etc/openvpn/up.sh (run when VPN connects)
#!/bin/bash
# Block all IPv6 except through VPN
ip6tables -I OUTPUT 1 ! -o tun0 -j DROP

# /etc/openvpn/down.sh (run when VPN disconnects)
#!/bin/bash
# Remove the blocking rule (or keep it — depends on policy)
ip6tables -D OUTPUT ! -o tun0 -j DROP
```

```ini
# Add to OpenVPN client config:
script-security 2
up /etc/openvpn/up.sh
down /etc/openvpn/down.sh
```

## Method 6: NetworkManager IPv6 Configuration

On Linux with NetworkManager:

```bash
# Disable IPv6 for a specific connection
nmcli connection modify "VPN Connection" ipv6.method disabled

# Or ignore IPv6 (link-local only, no routing)
nmcli connection modify "VPN Connection" ipv6.method link-local
```

## Testing for IPv6 Leaks

```bash
# Check your current IPv6 address
curl -6 https://ifconfig.co

# Use online leak tests
# https://ipleak.net
# https://ipv6leak.com
# https://browserleaks.com/ipv6

# Manual test: if you have IPv6, this should fail (blocked) or show VPN's IPv6
curl --connect-timeout 5 -6 https://ipv6.icanhazip.com
```

## Verifying Leak Prevention

```bash
# With VPN connected, try to reach IPv6
ping6 -c 2 2001:4860:4860::8888

# If leak prevention is working:
# - IPv6 is disabled: "ping6: connect: Network is unreachable"
# - IPv6 is blocked: no response
# - IPv6 is tunneled: works, but exits at VPN server IP
```

Preventing IPv6 leaks is essential for any VPN deployment where privacy or security depends on all traffic exiting through the VPN — blocking or tunneling IPv6 ensures complete coverage.
