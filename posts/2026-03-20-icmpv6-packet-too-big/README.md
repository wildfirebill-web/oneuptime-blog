# How to Handle ICMPv6 Packet Too Big in Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, Packet Too Big, PMTUD, IPv6, Socket Programming

Description: Handle ICMPv6 Packet Too Big messages in socket applications, update PMTU caches, and implement application-level PMTU Discovery for UDP applications.

## Introduction

ICMPv6 Packet Too Big (Type 2) messages notify a sending application that a packet exceeded the path MTU. For TCP, the kernel handles this automatically. For UDP applications, the application may need to handle these messages explicitly by reducing the packet size and retransmitting. Understanding how to receive and process PTB messages enables building reliable UDP-based protocols on IPv6.

## How the Kernel Delivers PTB to Applications

```text
ICMPv6 PTB delivery to applications:

TCP:
  Kernel handles PTB automatically
  Updates the PMTU cache
  Adjusts TCP MSS for the connection
  Application is unaware (transparent)

UDP (without IPV6_RECVPATHMTU):
  Kernel updates PMTU cache
  Subsequent sends may fail with EMSGSIZE
  Application receives send error on next oversized packet

UDP (with IPV6_RECVPATHMTU enabled):
  Kernel delivers PTB as ancillary data to recvmsg()
  Application can retrieve PTB information
  Application must reduce packet size explicitly
```

## Receiving PTB Notifications in Python

```python
import socket
import struct
import ctypes

def create_pmtu_aware_udp6_socket(bind_port: int) -> socket.socket:
    """
    Create a UDP/IPv6 socket that receives PMTU change notifications.
    """
    s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
    s.bind(('::', bind_port))

    # Enable PMTU discovery and notifications
    # IPV6_RECVPATHMTU = 60 (Linux)
    IPV6_RECVPATHMTU = 60
    s.setsockopt(socket.IPPROTO_IPV6, IPV6_RECVPATHMTU, 1)

    # Set IPV6_DONTFRAG: fails send with EMSGSIZE if packet exceeds PMTU
    IPV6_DONTFRAG = 62
    s.setsockopt(socket.IPPROTO_IPV6, IPV6_DONTFRAG, 1)

    return s

def send_with_pmtu_retry(s: socket.socket, data: bytes,
                          dest: tuple, pmtu: int = 1500) -> int:
    """
    Send UDP data with PMTU-aware retry.
    Reduces packet size if EMSGSIZE error received.
    """
    MAX_RETRIES = 5
    current_pmtu = pmtu
    IPV6_HEADER = 40
    UDP_HEADER = 8

    for attempt in range(MAX_RETRIES):
        max_payload = current_pmtu - IPV6_HEADER - UDP_HEADER
        if len(data) > max_payload:
            print(f"Payload {len(data)} bytes exceeds MTU {current_pmtu}, reducing")
            # For a real application, you'd need to chunk or wait
            data = data[:max_payload]

        try:
            return s.sendto(data, dest)
        except OSError as e:
            import errno
            if e.errno == errno.EMSGSIZE:
                # Packet too large for current PMTU
                # Get new PMTU from the route cache
                current_pmtu = max(1280, current_pmtu - 8)  # Conservative reduction
                print(f"EMSGSIZE: reducing PMTU estimate to {current_pmtu}")
            else:
                raise
    raise OSError("Max PMTU retries exceeded")
```

## Reading PTB MTU from Ancillary Data

```python
import socket
import struct

def receive_with_pmtu(s: socket.socket, bufsize: int = 4096):
    """
    Receive a UDP message and any PMTU change notifications in ancillary data.
    Returns (data, src_addr, new_pmtu_or_None)
    """
    # Increase receive buffer for ancillary data
    data, ancdata, flags, addr = s.recvmsg(bufsize, 1024)

    new_pmtu = None
    for cmsg_level, cmsg_type, cmsg_data in ancdata:
        # IPPROTO_IPV6 = 41, IPV6_PATHMTU = 61
        if cmsg_level == socket.IPPROTO_IPV6 and cmsg_type == 61:
            # IPV6_PATHMTU ancillary data structure:
            # struct ip6_mtuinfo { struct sockaddr_in6 ip6m_addr; uint32_t ip6m_mtu; }
            # The MTU is at offset 28 (24 bytes sockaddr_in6 + 4 bytes MTU)
            if len(cmsg_data) >= 32:
                mtu = struct.unpack_from("!I", cmsg_data, 28)[0]
                new_pmtu = mtu
                print(f"PMTU change notification: new MTU = {mtu}")

    return data, addr, new_pmtu
```

## Checking PMTU Before Sending

```bash
# Check cached PMTU for a destination before sending

ip -6 route get 2001:db8::server

# If an MTU entry is in the cache, the system has learned the PMTU
# Example output:
# 2001:db8::server via fe80::1 dev eth0 src 2001:db8::client
#   cache  expires 594sec mtu 1480

# Force flush PMTU cache to restart discovery
sudo ip -6 route flush cache

# Monitor PMTU changes in real-time
watch -n 1 'ip -6 route show cache | grep mtu'
```

## PTB and TCP Behavior

```bash
# TCP connections automatically handle PTB
# Verify TCP is respecting PMTU by watching for segment size reduction
sudo tcpdump -i eth0 -v "tcp and host 2001:db8::server" 2>&1 | \
    grep "length" | awk '{print $NF}' | sort -u
# Segment sizes should cluster at the PMTU-derived MSS

# If PTB arrives after connection established:
# Linux updates PMTU cache and reduces MSS for the connection
# No application action needed
```

## Conclusion

ICMPv6 Packet Too Big handling differs between TCP and UDP. TCP connections benefit from automatic kernel handling with no application involvement. UDP applications need to either tolerate EMSGSIZE errors when `IPV6_DONTFRAG` is set, or enable `IPV6_RECVPATHMTU` to receive explicit PTB notifications through ancillary data. The key implementation requirement: always check the PMTU cache before sending large UDP datagrams on new paths, and handle EMSGSIZE errors gracefully by reducing packet size and retransmitting.
