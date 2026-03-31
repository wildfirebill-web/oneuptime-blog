# How to Configure IPv6 Jumbograms on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Jumbograms, Linux, Jumbo Frame, HPC Networking

Description: Configure Linux to send and receive IPv6 jumbograms, set jumbo frame MTU on high-performance interfaces, and verify jumbogram support in the kernel.

## Introduction

Linux has supported IPv6 jumbograms since the early kernel 2.6 series. Enabling jumbograms on Linux requires appropriate hardware and driver support for large MTUs (typically ≥ 9000 bytes for "jumbo frames," and much larger for true jumbogram use cases). The configuration primarily involves setting the interface MTU to a value large enough to carry jumbogram-sized packets.

## Setting Up Jumbo Frames for Jumbogram Support

```bash
# Check current MTU and maximum supported MTU for an interface

ip link show eth0
ethtool eth0 | grep -i mtu

# Check if the NIC supports jumbo frames
ethtool eth0 | grep -i "max.*frame\|mtu"

# Set MTU to 9000 bytes (typical jumbo frame size)
sudo ip link set eth0 mtu 9000

# Verify the change
ip link show eth0 | grep mtu

# For true jumbogram support (up to 65535+ bytes), set MTU to max supported
# This is hardware-dependent; Infiniband or specialized NICs may support more
sudo ip link set ib0 mtu 65520  # Example for Infiniband

# Check the configured MTU in the kernel
cat /proc/sys/net/ipv6/conf/eth0/mtu
```

## Linux Kernel Jumbogram Support

```bash
# Check if the running kernel has IPv6 jumbogram support
grep -r "JUMBOGRAM\|RFC2675" /boot/config-$(uname -r) 2>/dev/null || \
    zgrep -i "jumbogram" /boot/config-$(uname -r).gz 2>/dev/null

# Check kernel jumbogram-related sysctl settings
sysctl -a 2>/dev/null | grep -i "ipv6.*jumbo"

# TCP buffer sizes for large transfers (important for jumbogram paths)
cat /proc/sys/net/core/rmem_max
cat /proc/sys/net/core/wmem_max

# For high-performance jumbogram paths, increase socket buffers
sudo sysctl -w net.core.rmem_max=134217728   # 128 MB receive buffer
sudo sysctl -w net.core.wmem_max=134217728   # 128 MB send buffer
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 67108864"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 67108864"
```

## Verifying Jumbogram Sending with Python

```python
import socket
import struct

def create_jumbogram_socket() -> socket.socket:
    """
    Create a raw IPv6 socket for sending jumbograms.
    Requires root privileges.
    """
    # RAW socket for IPv6
    s = socket.socket(socket.AF_INET6, socket.SOCK_RAW, socket.IPPROTO_RAW)

    # Set IPV6_HDRINCL to include our own IPv6 header
    s.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_HDRINCL, 1)

    # Increase socket buffer for large payloads
    s.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF, 10 * 1024 * 1024)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 10 * 1024 * 1024)

    return s

def check_interface_supports_jumbograms(interface: str,
                                         min_mtu: int = 65536) -> dict:
    """
    Check if an interface can support jumbograms.
    """
    import subprocess
    result = subprocess.run(
        ["ip", "link", "show", interface],
        capture_output=True, text=True
    )

    import re
    mtu_match = re.search(r'mtu (\d+)', result.stdout)
    current_mtu = int(mtu_match.group(1)) if mtu_match else 0

    return {
        "interface": interface,
        "current_mtu": current_mtu,
        "min_mtu_for_jumbograms": min_mtu,
        "supports_jumbograms": current_mtu >= min_mtu,
        "recommendation": (
            f"Set MTU to at least {min_mtu} for jumbogram support"
            if current_mtu < min_mtu
            else "Interface MTU adequate for jumbograms"
        )
    }

# Check common interfaces
for iface in ["eth0", "lo"]:
    result = check_interface_supports_jumbograms(iface)
    print(f"{result['interface']}: MTU={result['current_mtu']} - {result['recommendation']}")
```

## Performance Implications of Jumbograms

```bash
# Baseline performance test with standard MTU (1500 bytes)
iperf3 -6 -c 2001:db8::server -t 30 -P 4

# Performance test with jumbo frames (9000 byte MTU)
# First set both ends to MTU 9000
sudo ip link set eth0 mtu 9000
iperf3 -6 -c 2001:db8::server -t 30 -P 4 -l 8192

# Compare CPU usage and throughput
# Jumbograms reduce per-packet overhead:
# - Fewer interrupts from NIC
# - Less TCP/IP processing overhead per byte
# - Typically 10-30% improvement for bulk transfers on HPC links

# Check interrupt rate (lower is better with jumbograms)
watch -n 1 'cat /proc/interrupts | grep eth0'
```

## Persistent Jumbo Frame MTU Configuration

```bash
# Using systemd-networkd (/etc/systemd/network/10-eth0.network)
sudo tee /etc/systemd/network/10-eth0.network << 'EOF'
[Match]
Name=eth0

[Link]
MTUBytes=9000

[Network]
DHCP=yes
IPv6AcceptRA=yes
EOF

sudo systemctl restart systemd-networkd

# Using NetworkManager
nmcli connection modify "Wired connection 1" 802-3-ethernet.mtu 9000
nmcli connection up "Wired connection 1"

# Using /etc/network/interfaces (Debian)
# Add to eth0 stanza: mtu 9000
```

## Conclusion

Linux jumbogram support requires hardware that supports large MTUs and explicit MTU configuration on the interface. For practical jumbo frames (9000 bytes), standard server NICs with jumbo frame support are sufficient. For true IPv6 jumbograms beyond 65535 bytes, specialized hardware (InfiniBand, Cray, custom HPC interconnects) is required. In all cases, both endpoints and all intermediate links must be configured with the large MTU, and socket buffers should be increased to take full advantage of the throughput gains.
