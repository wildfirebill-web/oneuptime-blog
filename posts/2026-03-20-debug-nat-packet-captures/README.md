# How to Debug NAT Issues with Packet Captures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, Troubleshooting, tcpdump, Wireshark

Description: Learn how to use packet captures on both NAT interfaces to verify translations, diagnose failures, and trace packet flow through a NAT gateway.

## The Key to NAT Debugging: Capture Both Sides

To verify NAT is working correctly, capture on both the inside and outside interfaces simultaneously. The source IP should change between inside and outside captures.

```text
Inside capture (eth0): src=192.168.1.10 → dst=8.8.8.8
Outside capture (eth1): src=203.0.113.1 → dst=8.8.8.8  ← Translation confirmed
```

## Simultaneous Dual-Interface Capture

```bash
# Terminal 1: capture inside interface

sudo tcpdump -n -i eth0 -w /tmp/inside.pcap host 192.168.1.10

# Terminal 2: capture outside interface  
sudo tcpdump -n -i eth1 -w /tmp/outside.pcap host 203.0.113.1

# Generate traffic from inside host:
# ping 8.8.8.8 from 192.168.1.10

# Stop captures and compare
```

## Comparing Inside vs Outside

```bash
# Read inside capture
tcpdump -n -r /tmp/inside.pcap
# Shows: 192.168.1.10 → 8.8.8.8 and 8.8.8.8 → 192.168.1.10

# Read outside capture
tcpdump -n -r /tmp/outside.pcap
# Shows: 203.0.113.1 → 8.8.8.8 and 8.8.8.8 → 203.0.113.1
```

If outside capture shows nothing, NAT is not translating (check iptables rules, IP forwarding).

## Verifying Port Forwarding with Captures

Test port forward: external:80 → internal:192.168.1.10:80

```bash
# Capture on outside for incoming port 80
sudo tcpdump -n -i eth1 -w /tmp/ext80.pcap tcp port 80

# Capture on inside for forwarded traffic
sudo tcpdump -n -i eth0 -w /tmp/int80.pcap tcp port 80

# From external host: curl http://203.0.113.1:80

# Compare:
# Outside: dst=203.0.113.1:80 (incoming connection)
# Inside:  dst=192.168.1.10:80 (DNAT translated)
```

## Debugging with tcpdump Filters

```bash
# Capture all NAT-related traffic on outside interface
sudo tcpdump -n -i eth1 -v 'host 203.0.113.1'

# Capture ICMP to verify NAT for ping
sudo tcpdump -n -i eth0 icmp
sudo tcpdump -n -i eth1 icmp

# Capture TCP SYN packets (connection attempts) on both sides
sudo tcpdump -n -i eth0 'tcp[tcpflags] & tcp-syn != 0'
sudo tcpdump -n -i eth1 'tcp[tcpflags] & tcp-syn != 0'
```

## Wireshark: Follow NAT Sessions

```bash
# Capture on inside interface for a session
sudo tcpdump -i eth0 -w /tmp/session.pcap host 192.168.1.10 and host 8.8.8.8
```

In Wireshark:
1. Open the pcap
2. Right-click on a TCP packet → **Follow → TCP Stream**
3. View the complete conversation

## Using nftables/iptables Logging

```bash
# Add a logging rule BEFORE the NAT rule to see pre-NAT traffic
iptables -t nat -I PREROUTING 1 -j LOG --log-prefix "PRE-NAT: "

# Add logging in POSTROUTING to see post-NAT traffic
iptables -t nat -I POSTROUTING 1 -j LOG --log-prefix "POST-NAT: "

# View in system log
tail -f /var/log/kern.log | grep "PRE-NAT\|POST-NAT"
```

Remember to remove these log rules after debugging (they can be verbose):

```bash
iptables -t nat -D PREROUTING 1
iptables -t nat -D POSTROUTING 1
```

## Python: Analyzing Captured PCAP for NAT Translation

```python
from scapy.all import rdpcap, IP

def analyze_nat(pcap_file):
    packets = rdpcap(pcap_file)
    for pkt in packets:
        if IP in pkt:
            print(f"{pkt[IP].src}:{getattr(pkt.payload, 'sport', '-')} → "
                  f"{pkt[IP].dst}:{getattr(pkt.payload, 'dport', '-')}")

print("=== Inside Interface ===")
analyze_nat('/tmp/inside.pcap')
print("\n=== Outside Interface ===")
analyze_nat('/tmp/outside.pcap')
```

## Common Findings from Packet Captures

| Finding | What It Means |
|---------|--------------|
| Traffic on inside, none on outside | IP forwarding disabled or NAT rule missing |
| Traffic on outside with private src IP | SNAT/MASQUERADE rule missing |
| Traffic on outside, no reply | Firewall or routing issue on outside |
| Reply on outside, nothing on inside | conntrack entry expired or FORWARD rule missing |
| Port forward traffic not reaching inside | DNAT rule or FORWARD rule missing |

## Key Takeaways

- Always capture on both inside and outside interfaces to verify NAT translation.
- If outside shows no traffic, check IP forwarding and POSTROUTING rules.
- If inside shows no translated reply, check FORWARD chain and conntrack state.
- iptables LOG rules in nat table show pre/post-NAT IPs for debugging.

**Related Reading:**

- [How to Troubleshoot NAT Translation Issues](https://oneuptime.com/blog/post/2026-03-20-troubleshoot-nat-translation/view)
- [How to Troubleshoot NAT with Connection Tracking](https://oneuptime.com/blog/post/2026-03-20-nat-connection-tracking/view)
- [How to View the NAT Translation Table on Linux](https://oneuptime.com/blog/post/2026-03-20-view-nat-translation-table-linux/view)
