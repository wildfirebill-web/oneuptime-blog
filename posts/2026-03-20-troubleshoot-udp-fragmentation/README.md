# How to Troubleshoot UDP Fragmentation Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, Fragmentation, MTU, Linux, Networking, Troubleshooting

Description: Diagnose and fix UDP fragmentation problems caused by payloads exceeding the path MTU, including symptoms, detection methods, and configuration fixes.

## Introduction

UDP has no built-in fragmentation handling at the application layer. When a UDP payload plus IP and UDP headers exceeds the path MTU, one of two things happens: the kernel fragments the IP packet, or if the DF (Don't Fragment) bit is set, an ICMP Fragmentation Needed message is returned and the packet is dropped. Both outcomes cause problems — fragmentation increases loss probability and reassembly overhead; ICMP-blocked PMTUD creates silent delivery failures.

## Identifying Fragmentation Issues

```bash
# Check if fragmentation is occurring (kernel stats):
cat /proc/net/snmp | grep -i frag
# IpFragCreates: packets that were fragmented by this host
# IpFragFails: fragmentation failed (DF bit set and packet too large)
# IpReasmReqds: fragments received awaiting reassembly

# Or with nstat:
nstat -a | grep -i "frag\|reasmb"
# IpExtInMcastOctets etc

# Check for fragmentation in a packet capture:
tcpdump -i eth0 -n 'ip[6:2] & 0x3fff != 0'
# This matches packets with fragment offset > 0 OR MF bit set
# = fragmented packets
```

## Determine Your Effective MTU

```bash
# Standard Ethernet MTU is 1500 bytes
# UDP overhead:
# 20 bytes IP header + 8 bytes UDP header = 28 bytes overhead
# Max UDP payload without fragmentation = 1500 - 28 = 1472 bytes

# Test the maximum UDP payload size that passes without fragmentation:
# Note: ping uses ICMP (28 bytes overhead too), so ping -s 1472 tests same effective payload
ping -s 1472 -M do -c 3 10.20.0.5
# -M do: set DF bit (don't fragment)
# If succeeds: path MTU >= 1500

ping -s 1473 -M do -c 3 10.20.0.5
# If fails: path MTU is 1500 (1472 payload + 28 overhead = 1500)

# For VPN tunnels (add overhead for encapsulation):
# GRE: subtract 24 bytes (20 IP + 4 GRE): max payload = 1476 - 28 = 1448
# IPsec ESP: subtract ~50 bytes: max payload ~1422 bytes
```

## Detecting Fragmentation in Wireshark

```bash
# Wireshark display filters for fragmentation:

# Show fragmented packets:
ip.flags.mf == 1 or ip.frag_offset > 0

# Show only the first fragment (has MF bit and offset=0):
ip.flags.mf == 1 and ip.frag_offset == 0

# Show reassembled packets:
ip.reassembled_in

# Show large UDP packets likely to cause fragmentation:
udp and frame.len > 1472
```

## Fix: Limit UDP Payload Size

```bash
# Application-level fix: never send UDP payloads larger than safe size
# For standard Ethernet: keep payload <= 1472 bytes
# For VPN/tunnel environments: check actual path MTU with tracepath

tracepath -n 10.20.0.5
# Shows MTU at each hop

# Determine safe UDP payload size programmatically:
python3 -c "
import socket
import struct

def get_path_mtu(dst):
    # Use IP_PMTUDISC_PROBE to trigger MTU discovery
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.setsockopt(socket.IPPROTO_IP, socket.IP_MTU_DISCOVER, 3)  # IP_PMTUDISC_PROBE
    s.connect((dst, 9))
    mtu = s.getsockopt(socket.IPPROTO_IP, socket.IP_MTU)
    s.close()
    return mtu

mtu = get_path_mtu('10.20.0.5')
max_udp_payload = mtu - 28  # 20 IP + 8 UDP headers
print(f'Path MTU: {mtu}, Max UDP payload: {max_udp_payload}')
"
```

## Fix: Use IP_PMTUDISC_WANT

```bash
# Tell kernel to do path MTU discovery automatically
# Then use getsockopt IP_MTU to get the current path MTU

# Python:
# sock.setsockopt(socket.IPPROTO_IP, socket.IP_MTU_DISCOVER, 1)  # IP_PMTUDISC_WANT
# After sending and getting EMSGSIZE error: read current MTU
# sock.getsockopt(socket.IPPROTO_IP, socket.IP_MTU) → adjusted MTU

# Application should split payload into chunks <= path_mtu - 28
```

## Fix: Enable Fragmentation (Last Resort)

```bash
# If you can't control payload size, allow kernel fragmentation:
# (default on Linux: fragmentation is allowed if DF not set)

# In application: don't set IP_PMTUDISC_DO (which sets DF bit)
# sock.setsockopt(socket.IPPROTO_IP, socket.IP_MTU_DISCOVER, 0)  # IP_PMTUDISC_DONT

# But be aware: fragmentation on UDP increases loss rate
# A single lost fragment causes the entire datagram to fail reassembly
# (kernel cleans up incomplete fragments after ip_frag_time seconds)
sysctl net.ipv4.ipfrag_time  # Default: 30 seconds
```

## Conclusion

UDP fragmentation problems are best fixed by keeping UDP payloads below the path MTU minus 28 bytes. Use `tracepath` to find the actual path MTU, especially for VPN and tunnel environments where overhead reduces the effective MTU significantly. Monitor `IpFragCreates` and `IpFragFails` in kernel statistics to detect fragmentation in production. The most reliable fix is application-level payload size control with dynamic path MTU discovery via `IP_MTU_DISCOVER`.
