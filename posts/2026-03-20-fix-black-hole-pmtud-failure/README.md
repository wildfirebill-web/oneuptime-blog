# How to Fix Black Hole Router Issues Caused by PMTUD Failure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MTU, Black Hole, PMTUD, TCP, iptables, Linux, Networking, Troubleshooting

Description: Fix TCP connections that stall due to black hole routers blocking ICMP Fragmentation Needed messages, using MSS clamping, ICMP policy fixes, and path MTU override techniques.

## Introduction

Path MTU Discovery (PMTUD) depends on ICMP Type 3 Code 4 "Fragmentation Needed" messages being delivered back to the TCP sender. When a firewall blocks these messages, the sender never learns the smaller MTU and continues sending oversized packets that are silently dropped - creating a "black hole." The connection appears to work (handshake uses small packets) but data transfer stalls. Fixing this requires either unblocking ICMP or using MSS clamping as a bypass.

## Identify PMTUD Failure

```bash
# Symptom: large data transfers stall but small exchanges work

# Test: HTTP HEAD works but GET of large file hangs

# Step 1: Check if large ping reaches destination
ping -M do -s 1472 -c 3 10.20.0.5
# Timeout with no response = black hole candidate

# Step 2: Check if ICMP Fragmentation Needed is blocked
tcpdump -i eth0 -n 'icmp[0] = 3 and icmp[1] = 4' &
ping -M do -s 1472 -c 3 10.20.0.5
# If tcpdump shows NO ICMP type 3: firewall is blocking them
# If tcpdump shows ICMP type 3: PMTUD works, different issue

# Step 3: Confirm TCP stall pattern
curl -v http://10.20.0.5/large-file 2>&1 | head -30
# Watch: HTTP headers arrive (small), then stalls on body (large)

# Step 4: Test with smaller TCP MSS
curl --interface eth0 -v http://10.20.0.5/large-file &
# Capture SYN MSS:
tcpdump -n 'tcp[tcpflags] & tcp-syn != 0 and host 10.20.0.5' -v 2>&1 | grep mss
```

## Fix 1: Allow ICMP Fragmentation Needed (Preferred)

```bash
# This fixes the root cause: allow ICMP type 3 code 4 through firewalls

# On Linux iptables firewall:
iptables -I INPUT  -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -I OUTPUT -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -I FORWARD -p icmp --icmp-type fragmentation-needed -j ACCEPT

# Save rules persistently:
iptables-save > /etc/iptables/rules.v4

# Verify rule is in place:
iptables -L -n | grep -A 2 "icmp"

# On cloud security groups (AWS):
# Inbound rule: Allow ICMP type 3 from 0.0.0.0/0
# aws ec2 authorize-security-group-ingress \
#   --group-id sg-xxxx \
#   --protocol icmp \
#   --port -1 \
#   --cidr 0.0.0.0/0

# Test: ICMP should now arrive when large packet hits smaller MTU:
tcpdump -i eth0 -n 'icmp[0] = 3 and icmp[1] = 4'
ping -M do -s 1472 -c 3 10.20.0.5
```

## Fix 2: TCP MSS Clamping (Works Without ICMP)

```bash
# MSS clamping rewrites TCP SYN MSS to prevent oversized segments
# Works even if ICMP is blocked - prevents the problem before it starts

# On the router/gateway between networks:
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# If you know the exact path MTU (e.g., 1400):
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --set-mss 1360  # path-MTU (1400) - 40

# Apply to outbound AND inbound traffic:
iptables -t mangle -A OUTPUT -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# Verify:
iptables -t mangle -L -v -n | grep TCPMSS

# Test: TCP connections should now complete large transfers
curl http://10.20.0.5/large-file -o /dev/null --progress-bar
```

## Fix 3: Override PMTUD on Sockets

```bash
# Disable PMTUD entirely for specific connections:
# This allows fragmentation instead of blocking

# System-wide (not recommended for production):
sysctl -w net.ipv4.ip_no_pmtu_disc=1
# 0 = PMTUD enabled (default)
# 1 = PMTUD disabled (always fragment, never set DF)

# Per-socket in application code:
# IP_MTU_DISCOVER with IP_PMTUDISC_DONT

# Or set a fixed MTU hint:
sysctl -w net.ipv4.route.min_pmtu=576
# Forces PMTUD to cache at least 576 as minimum

# View current PMTU cache:
ip route show cache
# Shows per-destination MTU cache entries

# Clear PMTU cache for specific destination:
ip route flush cache 10.20.0.5
# Forces fresh PMTU discovery
```

## Fix 4: Reduce Interface MTU

```bash
# If you control the bottleneck: just reduce MTU to match path

# Find actual path MTU:
tracepath -n 10.20.0.5
# Read the pmtu value from output

# Set interface MTU to match path:
PMTU=$(tracepath -n 10.20.0.5 | grep "Resume" | \
  grep -oP 'pmtu \K[0-9]+' || echo "1500")
ip link set eth0 mtu $PMTU
echo "Set interface MTU to $PMTU"

# All connections from this host will now fit the path MTU
ip link show eth0 | grep mtu
```

## Verify the Fix Works

```bash
# After applying fix, run comprehensive test:

echo "=== Testing MTU Black Hole Fix ==="

# Test 1: Large ping should work now
ping -M do -s 1472 -c 3 10.20.0.5 && echo "PASS: Large ping works" || echo "FAIL: Large ping still drops"

# Test 2: Large file download should complete
wget -O /dev/null -q --show-progress http://10.20.0.5/large-file && \
  echo "PASS: Large HTTP download works" || \
  echo "FAIL: Large HTTP download still stalls"

# Test 3: No more PMTUD-related ICMP blocked
tcpdump -i eth0 -c 5 -n 'icmp[0] = 3 and icmp[1] = 4' 2>/dev/null | \
  grep -c "ICMP" && echo "INFO: ICMP Frag Needed messages are flowing"

# Test 4: Kernel PMTU cache shows correct MTU:
ping -c 1 -q 10.20.0.5 > /dev/null
ip route show cache 10.20.0.5 | grep -oP 'mtu \K[0-9]+'
```

## Conclusion

PMTUD black holes are fixed by either allowing ICMP Type 3 Code 4 through all firewalls on the path (the proper fix) or applying TCP MSS clamping on your edge router (the practical workaround). MSS clamping is preferred when you don't control all firewalls: it prevents TCP from ever sending segments that would exceed the path MTU. Apply `--clamp-mss-to-pmtu` on FORWARD and OUTPUT chains. After applying the fix, verify with `wget` or `curl` of large files - these expose black holes better than ping does.
