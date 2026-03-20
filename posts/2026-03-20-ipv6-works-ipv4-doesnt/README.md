# How to Troubleshoot IPv6 Connectivity Works But IPv4 Does Not

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPv4, Troubleshooting, Connectivity, Dual-Stack, Network Debugging

Description: Diagnose and fix situations where IPv6 connectivity is working but IPv4 is broken, including common causes like missing IPv4 routes, DHCP failures, NAT misconfigurations, and ISP IPv4 issues.

## Introduction

When IPv6 works but IPv4 does not, the root cause is typically in the IPv4 layer specifically — IPv6 and IPv4 use separate address assignment, routing, and firewall mechanisms. Common causes include DHCP failure (no IPv4 address assigned), missing IPv4 default route, NAT table overflow, ISP IPv4 outage, or firewall rules blocking IPv4 but not IPv6. This guide provides systematic diagnosis and fixes.

## Diagnostic Checklist

```bash
#!/bin/bash
echo "=== IPv4 vs IPv6 Comparison Diagnostic ==="

echo ""
echo "--- IPv4 Address Assignment ---"
ip -4 addr show | grep "inet " | grep -v "127.0.0.1"
# Should show: inet x.x.x.x/xx

echo ""
echo "--- IPv6 Address Assignment ---"
ip -6 addr show | grep "inet6.*scope global"
# Should show: inet6 xxxx::/xx scope global

echo ""
echo "--- IPv4 Default Route ---"
ip -4 route show default
# Should show: default via x.x.x.x dev eth0

echo ""
echo "--- IPv6 Default Route ---"
ip -6 route show default
# Should show: default via xxxx:: dev eth0

echo ""
echo "--- IPv4 Gateway Ping ---"
GW4=$(ip -4 route show default | awk '{print $3}' | head -1)
[ -n "$GW4" ] && ping -c 2 "$GW4" || echo "No IPv4 gateway found"

echo ""
echo "--- IPv6 Gateway Ping ---"
GW6=$(ip -6 route show default | awk '{print $3}' | head -1)
[ -n "$GW6" ] && ping6 -c 2 "$GW6" || echo "No IPv6 gateway found"

echo ""
echo "--- IPv4 DNS Test ---"
nslookup google.com 8.8.8.8 2>/dev/null | grep "Address" || echo "IPv4 DNS failed"

echo ""
echo "--- IPv6 DNS Test ---"
nslookup google.com 2001:4860:4860::8888 2>/dev/null | grep "Address" || echo "IPv6 DNS failed"
```

## Fix: No IPv4 Address (DHCP Failure)

```bash
# Symptom: No inet address on interface (only link-local IPv4 169.254.x.x or nothing)
ip -4 addr show eth0

# Fix 1: Request DHCP manually
sudo dhclient eth0
# or
sudo dhcpcd eth0

# Fix 2: Check DHCP client status
systemctl status dhclient
systemctl status dhcpcd
journalctl -u dhclient -n 20

# Fix 3: Release and renew DHCP
sudo dhclient -r eth0  # Release
sudo dhclient eth0     # Request new lease

# Fix 4: Check DHCP server reachability
sudo dhclient -v eth0  # Verbose DHCP negotiation
# Look for: DHCPDISCOVER, DHCPOFFER, DHCPREQUEST, DHCPACK
```

## Fix: IPv4 Default Route Missing

```bash
# Symptom: IPv4 address assigned but no internet access
ip -4 route show default
# Empty or missing

# Fix: Add default route manually
sudo ip route add default via $(ip -4 route show | grep "default" | awk '{print $3}')
# Or specify gateway directly
sudo ip route add default via 192.168.1.1 dev eth0

# Make persistent (Debian/Ubuntu)
cat >> /etc/network/interfaces << 'EOF'
post-up route add default gw 192.168.1.1
EOF

# systemd-networkd
cat > /etc/systemd/network/eth0.network << 'EOF'
[Match]
Name=eth0

[Network]
DHCP=yes
IPv6AcceptRA=yes
EOF
```

## Fix: NAT/Firewall IPv4 Issue

```bash
# Check iptables for IPv4 blocking rules
sudo iptables -L -n | grep -E "DROP|REJECT"

# Check if NAT masquerade is configured for IPv4 outbound
sudo iptables -t nat -L POSTROUTING -n

# If NAT is missing and needed:
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Check IPv4 forwarding (for routing hosts)
cat /proc/sys/net/ipv4/ip_forward
# Should be 1 if forwarding is needed

# Enable IPv4 forwarding
sudo sysctl -w net.ipv4.ip_forward=1
```

## Fix: ISP IPv4 vs IPv6 Outage

```bash
# Test IPv4 and IPv6 independently
# IPv4 test
curl -4 -s --connect-timeout 5 http://ipv4.google.com/ && echo "IPv4 internet: OK" || echo "IPv4 internet: FAIL"

# IPv6 test
curl -6 -s --connect-timeout 5 http://ipv6.google.com/ && echo "IPv6 internet: OK" || echo "IPv6 internet: FAIL"

# Traceroute IPv4 to find where packets are dropped
traceroute -4 8.8.8.8

# Compare with IPv6
traceroute6 2001:4860:4860::8888

# If IPv4 fails at ISP hop, contact ISP about IPv4 outage
# IPv6 may use different infrastructure (native vs tunneled)
```

## Conclusion

When IPv6 works but IPv4 fails, diagnose in order: check IPv4 address assignment (DHCP failure is common), verify IPv4 default route exists, check iptables rules for IPv4-specific blocks, verify NAT masquerade is configured for IPv4 outbound, and test if the issue is at the ISP level. IPv6 and IPv4 use completely separate routing, DHCP, and firewall paths — a failure in one does not affect the other. The most common cause is DHCP failure combined with working IPv6 SLAAC, which assigns IPv6 automatically without DHCP.
