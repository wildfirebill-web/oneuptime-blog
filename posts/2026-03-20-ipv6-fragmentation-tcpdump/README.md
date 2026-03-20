# How to Analyze IPv6 Fragmentation with tcpdump

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Fragmentation, tcpdump, Packet Analysis, Debugging

Description: Use tcpdump to capture and analyze IPv6 fragmented packets, identify fragment headers, trace reassembly sequences, and diagnose fragmentation-related issues.

## Introduction

tcpdump is the first tool to reach for when debugging IPv6 fragmentation issues. Its BPF filters can isolate fragmented traffic, and its verbose output mode decodes Fragment Header fields. Understanding how to filter for and read fragment information in tcpdump output is essential for diagnosing PMTU failures, packet drops, and reassembly issues.

## Filtering IPv6 Fragments in tcpdump

```bash
# Capture all IPv6 packets with a Fragment Header

# (Next Header field at byte 6 of IPv6 header == 44)
sudo tcpdump -i eth0 "ip6[6] == 44"

# Capture fragments with "More Fragments" flag set (not the last fragment)
# Fragment offset/flags at bytes 42-43, More flag is bit 0 of byte 43
sudo tcpdump -i eth0 "ip6[6] == 44 and (ip6[43] & 1) == 1"

# Capture last fragments (More = 0, Offset != 0)
# These complete a fragmented sequence
sudo tcpdump -i eth0 "ip6[6] == 44 and (ip6[43] & 1) == 0"

# Capture ICMPv6 Packet Too Big (type 2) - indicates PMTU issue
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 2"

# Combine: capture fragments AND Packet Too Big messages
sudo tcpdump -i eth0 "ip6[6] == 44 or (icmp6 and ip6[40] == 2)"

# Save to file for Wireshark analysis
sudo tcpdump -i eth0 -w /tmp/fragments.pcap "ip6[6] == 44"
```

## Reading tcpdump Fragment Output

```bash
# Use -v for verbose output that shows extension header fields
sudo tcpdump -i eth0 -v "ip6[6] == 44"

# Example verbose output for a fragmented packet:
# 14:23:01.123456 IP6 (hlim 64, next-header Fragment (44) payload length: 1452)
#   2001:db8::1 > 2001:db8::2: frag (0x12345678|0) 1448
#   2001:db8::1 > 2001:db8::2: frag (0x12345678|1448) 556

# Interpreting the output:
# (0x12345678|0)    → Identification=0x12345678, Offset=0 (first fragment)
# 1448              → Data length in this fragment
# (0x12345678|1448) → Same Identification, Offset=1448 (second fragment)
# 556               → Data length in this fragment

# Use -vv for even more detail
sudo tcpdump -i eth0 -vv "ip6[6] == 44" | head -50
```

## Tracing a Complete Fragmentation Sequence

```bash
# Capture all packets between two hosts and filter for fragments
sudo tcpdump -i eth0 -v \
    "host 2001:db8::1 and host 2001:db8::2 and ip6[6] == 44"

# Check if Packet Too Big messages precede the fragments
# (source learns new PMTU and starts fragmenting)
sudo tcpdump -i eth0 -v \
    "(host 2001:db8::1 or host 2001:db8::2) and \
     (ip6[6] == 44 or (icmp6 and ip6[40] == 2))"

# Show timestamps with microsecond precision
sudo tcpdump -i eth0 -tttt -v "ip6[6] == 44"

# Count fragments per second
sudo tcpdump -i eth0 -q "ip6[6] == 44" 2>&1 | \
    awk '{count++} END {print count " fragments captured"}'
```

## Analyzing Fragment Statistics

```bash
# Watch fragment rate in real-time
sudo tcpdump -i eth0 -q "ip6[6] == 44" 2>&1 | \
    awk 'NR%100==0 {print NR " fragments seen"}'

# Check kernel fragment statistics
watch -n 1 'cat /proc/net/snmp6 | grep -E "ReasmR|Frag"'
# Ip6ReasmReqds:    reassembly attempts
# Ip6ReasmOKs:      successful reassemblies
# Ip6ReasmFails:    failed reassemblies
# Ip6FragCreates:   fragments created by this host
# Ip6FragOKs:       successful fragmentations
# Ip6FragFails:     fragmentation failures

# Test fragmentation by sending a large UDP packet
# (UDP doesn't do PMTUD by default, so it will fragment)
python3 -c "
import socket
s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
s.sendto(b'X' * 3000, ('2001:db8::1', 12345))
"
# Then capture on the sending interface to see fragments
```

## Script to Parse Fragment Sequences

```python
import subprocess
import re
from collections import defaultdict

def analyze_fragments(interface: str = "eth0", count: int = 100) -> dict:
    """
    Capture and analyze IPv6 fragment sequences.
    Returns statistics on fragmentation activity.
    """
    result = subprocess.run(
        ["tcpdump", "-i", interface, "-v", "-c", str(count),
         "ip6[6] == 44"],
        capture_output=True, text=True, timeout=30
    )

    sequences = defaultdict(list)
    # Parse fragment lines: "frag (0xID|OFFSET) LENGTH"
    pattern = r'frag \(0x([0-9a-f]+)\|(\d+)\) (\d+)'
    for line in result.stdout.split('\n'):
        m = re.search(pattern, line)
        if m:
            ident, offset, length = m.group(1), int(m.group(2)), int(m.group(3))
            sequences[ident].append({"offset": offset, "length": length})

    stats = {
        "total_fragments": sum(len(v) for v in sequences.values()),
        "unique_sequences": len(sequences),
        "sequences": dict(sequences),
    }
    return stats

# Run analysis
# stats = analyze_fragments("eth0", 50)
# print(f"Captured {stats['total_fragments']} fragments in {stats['unique_sequences']} sequences")
```

## Conclusion

tcpdump's BPF filter `ip6[6] == 44` is the essential selector for IPv6 fragmented packets. Combined with `-v` for verbose output, it reveals the Fragment Header's Identification value and offset for each fragment. When troubleshooting PMTU issues, always capture both Fragment Header packets and ICMPv6 Packet Too Big messages simultaneously to understand the full picture. The kernel's `/proc/net/snmp6` counters provide aggregate statistics without needing to capture every packet.
