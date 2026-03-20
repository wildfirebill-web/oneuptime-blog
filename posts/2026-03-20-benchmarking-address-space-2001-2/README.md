# How to Understand the Benchmarking Address Space (2001:2::/48)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Benchmarking, 2001:2::/48, RFC 5180, Testing, Lab

Description: Understand the IPv6 Benchmarking address space 2001:2::/48 (RFC 5180), its purpose for network device testing, and how to use it correctly in lab environments.

## Introduction

`2001:2::/48` is the IPv6 Benchmarking address space defined in RFC 5180. It is reserved for use in benchmark testing of network devices and links. Traffic from this prefix must not appear on the public internet. Using it in benchmarks ensures test traffic is clearly identifiable and does not conflict with real addresses.

## Key Properties

| Property | Value |
|---|---|
| Prefix | 2001:2::/48 |
| RFC | RFC 5180 |
| Source | True (used in tests) |
| Destination | True (used in tests) |
| Forwardable | Yes (within test environment) |
| Globally Reachable | No |

## Using 2001:2::/48 in iperf3 Benchmarks

```bash
# Assign benchmarking addresses to test interfaces

ip -6 addr add 2001:2::1/48 dev eth0   # Source device
ip -6 addr add 2001:2::2/48 dev eth0   # Destination device

# Run iperf3 benchmark using benchmarking addresses
# Server
iperf3 -s -6 -B 2001:2::2

# Client
iperf3 -c 2001:2::2 -6 -t 60 -P 8 -B 2001:2::1

# Benefits:
# 1. Tcpdump filter makes test traffic easy to isolate
# 2. No risk of interfering with production traffic
# 3. Clearly marked as benchmark traffic in logs
```

## RFC 5180 Benchmark Methodology

```bash
# RFC 5180 specifies these test configurations for IPv6 benchmarking:

# 1. Throughput test
# Increase offered load until packet loss occurs (RFC 2544 methodology)
# Use 2001:2::/48 as source/destination

# 2. Frame sizes to test (RFC 2544):
FRAME_SIZES=(64 128 256 512 1024 1280 1518)

for SIZE in "${FRAME_SIZES[@]}"; do
  echo "Testing frame size: $SIZE bytes"
  iperf3 -c 2001:2::2 -6 -t 10 -l $SIZE --length $SIZE
done

# 3. Latency test at different load levels
for RATE in "100M" "500M" "1G"; do
  echo "Testing at $RATE"
  iperf3 -c 2001:2::2 -6 -u -b $RATE -t 10
done
```

## Python: Generate Benchmark Test Addresses

```python
import ipaddress

BENCH_BLOCK = ipaddress.IPv6Network("2001:2::/48")

def generate_bench_pairs(count: int) -> list:
    """Generate source/destination pairs within benchmarking space."""
    hosts = list(BENCH_BLOCK.hosts())
    pairs = []
    for i in range(0, min(count * 2, len(hosts)), 2):
        pairs.append((str(hosts[i]), str(hosts[i + 1])))
    return pairs

# Generate 5 test pairs
for src, dst in generate_bench_pairs(5):
    print(f"src={src} dst={dst}")

# Validate that an address is in benchmarking space
def is_benchmarking(addr_str: str) -> bool:
    try:
        addr = ipaddress.IPv6Address(addr_str)
        return addr in BENCH_BLOCK
    except ValueError:
        return False

print(is_benchmarking("2001:2::1"))        # True
print(is_benchmarking("2001:2:0:1::"))     # True
print(is_benchmarking("2001:2:1::"))       # False (outside /48)
print(is_benchmarking("2001:db8::1"))      # False (documentation)
```

## Filtering Benchmarking Traffic

```bash
# Ensure benchmarking traffic never leaves your test environment
ip6tables -A FORWARD -s 2001:2::/48 -o eth-external -j DROP
ip6tables -A FORWARD -d 2001:2::/48 -o eth-external -j DROP

# Log benchmarking traffic for analysis
ip6tables -A INPUT -s 2001:2::/48 -j LOG --log-prefix "BENCH: "
```

## Conclusion

The `2001:2::/48` benchmarking space provides a clearly reserved prefix for network device testing per RFC 5180. Always use it for lab throughput tests and never allow it to leak to the internet. The benchmarking address family makes test traffic easy to filter in pcap analysis. Use OneUptime to schedule benchmark runs and track performance trends over time.
