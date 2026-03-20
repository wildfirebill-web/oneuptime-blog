# How to Generate IPv6 Traffic with TRex and Scapy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TRex, Scapy, IPv6, Traffic Generation, Performance Testing, Lab

Description: Generate synthetic IPv6 traffic for testing with Cisco TRex for high-rate performance tests and Scapy for protocol-level packet crafting.

## Scapy IPv6 Packet Generation

Scapy provides Python-based packet crafting for protocol testing:

```python
from scapy.all import *
from scapy.layers.inet6 import *

# Send a single ICMPv6 ping
def send_icmpv6_ping(src, dst, count=3):
    pkt = IPv6(src=src, dst=dst) / ICMPv6EchoRequest(id=0x1234, seq=1)
    responses = []

    for i in range(count):
        pkt.seq = i + 1
        resp = sr1(pkt, timeout=2, verbose=False)
        if resp:
            print(f"Reply from {resp[IPv6].src}: seq={i+1}")
            responses.append(resp)
        else:
            print(f"Timeout: seq={i+1}")

    return responses

send_icmpv6_ping("2001:db8::1", "2001:db8::2")
```

## High-Rate IPv6 Packet Generation with Scapy

```python
from scapy.all import *
from scapy.layers.inet6 import *
import time

def generate_ipv6_udp_flood(src_net, dst, port, pps, duration):
    """Generate UDP packets at a target packets-per-second rate"""

    import ipaddress
    network = ipaddress.IPv6Network(src_net)
    hosts = list(network.hosts())

    packet_count = 0
    start = time.time()
    interval = 1.0 / pps

    print(f"Sending {pps} pkt/s to {dst}:{port} for {duration}s")

    while time.time() - start < duration:
        # Rotate source address to simulate multiple clients
        src = str(hosts[packet_count % len(hosts)])
        pkt = (IPv6(src=src, dst=dst) /
               UDP(sport=RandShort(), dport=port) /
               Raw(b'X' * 64))

        sendp(Ether() / pkt, iface="eth0", verbose=False)
        packet_count += 1
        time.sleep(interval)

    elapsed = time.time() - start
    print(f"Sent {packet_count} packets in {elapsed:.1f}s ({packet_count/elapsed:.0f} pkt/s)")

# Generate 1000 pkt/s for 10 seconds
generate_ipv6_udp_flood("2001:db8::/64", "2001:db8::server", 9000, 1000, 10)
```

## TRex IPv6 Traffic Profile

Cisco TRex is a stateless traffic generator capable of line-rate packet generation:

```python
# trex_ipv6_profile.py — TRex traffic profile for IPv6

from trex_stl_lib.api import *

class IPv6Profile:
    def get_streams(self, direction=0, **kwargs):
        # IPv6 UDP stream
        pkt = STLPktBuilder(pkt=
            Ether() /
            IPv6(src="2001:db8:1::1", dst="2001:db8:2::1") /
            UDP(sport=1024, dport=5001) /
            Raw(b'P' * 64)
        )

        return [
            STLStream(
                packet=pkt,
                mode=STLTXCont(pps=1_000_000),  # 1 Mpps
                flow_stats=STLFlowLatencyStats(pg_id=0),
            )
        ]

def register():
    return IPv6Profile()
```

```bash
# Start TRex server
trex-console

# Load and start IPv6 profile
tui> start -f trex_ipv6_profile.py -p 0 --force

# View statistics
tui> stats

# Stop
tui> stop -a
```

## NDP Stress Testing with Scapy

```python
from scapy.all import *
from scapy.layers.inet6 import *

def send_neighbor_solicitations(target_addr, iface, count=100):
    """Send multiple NDP Neighbor Solicitations to stress-test NDP cache"""

    # Build solicited-node multicast address
    last24 = target_addr.replace(':', '')[-6:]
    mcast = f"ff02::1:ff{last24[:2]}:{last24[2:]}"

    pkt = (Ether(dst="33:33:ff:00:00:01") /
           IPv6(src="fe80::1", dst=mcast) /
           ICMPv6ND_NS(tgt=target_addr) /
           ICMPv6NDOptSrcLLAddr(lladdr="52:54:00:11:22:33"))

    print(f"Sending {count} NS to {target_addr}")
    for i in range(count):
        sendp(pkt, iface=iface, verbose=False)
        time.sleep(0.01)

    print("Done. Check NDP cache with: ip -6 neigh show")

send_neighbor_solicitations("2001:db8::1", "eth0", 50)
```

## TCP Connection Rate Tester

```python
import socket
import time
import threading

def tcp_connect_rate(host, port, duration, workers=50):
    """Measure IPv6 TCP connection rate"""

    connected = [0]
    failed = [0]
    stop = [False]

    def worker():
        while not stop[0]:
            try:
                s = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
                s.settimeout(2)
                s.connect((host, port, 0, 0))
                s.close()
                connected[0] += 1
            except Exception:
                failed[0] += 1

    threads = [threading.Thread(target=worker) for _ in range(workers)]
    for t in threads: t.start()

    time.sleep(duration)
    stop[0] = True
    for t in threads: t.join()

    total = connected[0] + failed[0]
    print(f"TCP connect rate to [{host}]:{port}")
    print(f"  Total: {total} in {duration}s ({total/duration:.0f} conn/s)")
    print(f"  Success: {connected[0]}, Failed: {failed[0]}")

tcp_connect_rate("2001:db8::1", 80, 10, workers=20)
```

## Conclusion

Scapy is the most flexible tool for IPv6 traffic generation in test labs — it supports any packet structure at any rate up to ~100k pkt/s on a modern CPU. TRex enables line-rate (100 Gbps+) IPv6 testing for performance benchmarking and capacity planning. NDP stress tests using Scapy ICMPv6ND packets validate NDP cache behavior under load. TCP connection rate tests measure server IPv6 stack performance. Combine these tools with network namespace topologies for comprehensive IPv6 stack testing.
