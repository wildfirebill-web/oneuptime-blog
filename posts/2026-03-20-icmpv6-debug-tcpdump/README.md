# How to Debug ICMPv6 Issues with tcpdump

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, tcpdump, Debugging, Packet Analysis, IPv6

Description: Use tcpdump filters and output options to capture and analyze ICMPv6 messages, debug NDP, PMTU Discovery, and connectivity issues from the command line.

## Introduction

tcpdump is the primary command-line tool for ICMPv6 debugging. Unlike Wireshark, tcpdump is available on virtually every Linux/BSD system without a GUI, making it essential for server-side debugging. Its BPF filter syntax for ICMPv6 uses the IPv6 header bytes directly, while the verbose `-v` and `-vv` flags decode ICMPv6 message content. This guide covers filters and techniques for the most common ICMPv6 debugging scenarios.

## Basic ICMPv6 Capture Filters

```bash
# Capture all ICMPv6 traffic

sudo tcpdump -i eth0 "icmp6"

# Capture specific ICMPv6 types using byte offsets:
# ip6[40] = ICMPv6 Type field
# ip6[41] = ICMPv6 Code field

# Packet Too Big (Type 2)
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 2"

# Destination Unreachable (Type 1)
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 1"

# Port Unreachable (Type 1, Code 4)
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 1 and ip6[41] == 4"

# Time Exceeded (Type 3)
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 3"

# Echo Request and Reply (ping6)
sudo tcpdump -i eth0 "icmp6 and (ip6[40] == 128 or ip6[40] == 129)"

# NDP messages (Types 133-137)
sudo tcpdump -i eth0 \
    "icmp6 and ip6[40] >= 133 and ip6[40] <= 137"

# Neighbor Solicitation and Advertisement only
sudo tcpdump -i eth0 \
    "icmp6 and (ip6[40] == 135 or ip6[40] == 136)"

# Router Advertisement only
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 134"
```

## Verbose ICMPv6 Output

```bash
# -v: show ICMPv6 type, code, and key fields
sudo tcpdump -i eth0 -v "icmp6"

# -vv: maximum detail (shows full NDP options, MTU values, etc.)
sudo tcpdump -i eth0 -vv "icmp6 and ip6[40] == 134"
# Example RA output:
# 14:23:45.123 IP6 (hlim 255, next-header ICMPv6 (58) payload length: 56)
#   fe80::1 > ff02::1: [icmp6 sum ok] ICMP6, router advertisement, length 56
#   hop limit 64, Flags [other stateful], pref medium, router lifetime 1800s,
#   reachable time 0ms, retrans time 0ms
#   prefix info option (3), length 32 (4): 2001:db8::/64, Flags [onlink,auto]
#     valid time 2592000s, pref. time 604800s

# -nn: don't resolve hostnames or ports (faster, shows raw addresses)
sudo tcpdump -i eth0 -nn -v "icmp6"
```

## Debugging PMTU Issues

```bash
# Watch for PTB messages arriving at this host
# Save to file for later analysis if needed
sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 2" | tee /tmp/ptb-capture.txt

# Watch both PTB and large outgoing packets simultaneously
# (Terminal 1: watch for PTB)
sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 2"

# (Terminal 2: watch packet sizes)
sudo tcpdump -i eth0 "host 2001:db8::server" | \
    awk '{ for(i=1; i<=NF; i++) if ($i ~ /length/) print $(i+1) }' | \
    sort -n | uniq -c | sort -rn | head -20

# Test PMTUD in real-time
# Send incrementally larger packets and watch for PTB
for size in 576 1000 1200 1280 1400 1480 1500; do
    echo -n "Testing MTU $size: "
    result=$(ping6 -c 1 -M do -s $((size-48)) -W 2 2001:db8::server 2>&1)
    if echo "$result" | grep -q "1 received"; then
        echo "OK"
    elif echo "$result" | grep -q "too long\|mtu"; then
        echo "TOO BIG (PTB)"
    else
        echo "TIMEOUT (possible black hole)"
    fi
done
```

## Debugging NDP (Neighbor Discovery)

```bash
# Watch complete NDP exchange for address resolution
sudo tcpdump -i eth0 -vv \
    "icmp6 and (ip6[40] == 135 or ip6[40] == 136)"

# Check for DAD (Duplicate Address Detection)
# DAD uses :: as source address and solicited-node multicast as dest
sudo tcpdump -i eth0 -v \
    "icmp6 and ip6[40] == 135 and src :: "
# If you see NS from :: : DAD is happening (new address being configured)

# Check if your Neighbor Advertisements are being sent
sudo tcpdump -i eth0 -v \
    "icmp6 and ip6[40] == 136 and src 2001:db8::your_address"

# Debug why neighbor is unreachable (watch NS with no NA response)
sudo timeout 10 tcpdump -i eth0 -v \
    "icmp6 and (ip6[40] == 135 or ip6[40] == 136) and \
    (host 2001:db8::neighbor)"
# If you see NS but no NA within 1 second: neighbor is down or not responding
```

## Debugging traceroute6 with tcpdump

```bash
# Capture traceroute6 Time Exceeded replies
# Run in parallel with traceroute6 execution
sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 3 and ip6[41] == 0"

# Capture the full traceroute6 exchange
sudo tcpdump -i eth0 -v \
    "(icmp6 and ip6[40] == 3) or (udp and dst portrange 33434-33500)"

# Count Time Exceeded by source (shows how many hops replied)
sudo tcpdump -i eth0 -nn "icmp6 and ip6[40] == 3" 2>&1 | \
    awk '{print $3}' | sort | uniq -c | sort -rn
```

## Conclusion

tcpdump's BPF filter `"icmp6 and ip6[40] == <type>"` is the fundamental selector for specific ICMPv6 messages by type. Combining with `ip6[41] == <code>` allows filtering by code within a type. The `-v` and `-vv` flags provide detailed output for NDP option decoding, RA flags, and MTU values in Packet Too Big messages. For PMTU debugging, always watch for Type 2 messages. For connectivity failures, always check Type 1 (Destination Unreachable) and watch for NS/NA pairs to ensure neighbor resolution is working.
