# How to Implement IPv4 Multicast for Group Communication in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, Multicast, UDP, Networking, Group Communication

Description: Learn how to implement IPv4 multicast group communication in Python using UDP sockets, with multicast sender and receiver examples and IGMP group membership management.

## IPv4 Multicast Address Ranges

| Range | Scope | Examples |
|-------|-------|---------|
| 224.0.0.0/24 | Link-local | OSPF (224.0.0.5), mDNS (224.0.0.251) |
| 224.0.1.0–238.255.255.255 | Internet (global) | 224.1.2.3 |
| 239.0.0.0/8 | Administratively scoped (private) | 239.255.0.1 |

For application use, choose an address in the `239.x.x.x` range.

## Multicast Sender

```python
import socket
import struct
import time
import json

MCAST_GROUP = "239.255.0.1"
MCAST_PORT  = 5007
TTL         = 2    # hops - 1 = same subnet, >1 = multiple hops

def multicast_sender():
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

    # Set TTL (time-to-live / hop limit for multicast)
    sock.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_TTL,
                    struct.pack("b", TTL))

    # Optionally bind to a specific local interface
    # sock.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_IF,
    #                 socket.inet_aton("192.168.1.10"))

    print(f"Sending multicast to {MCAST_GROUP}:{MCAST_PORT}")

    for i in range(10):
        msg = json.dumps({"seq": i, "msg": f"Hello group #{i}"})
        sock.sendto(msg.encode(), (MCAST_GROUP, MCAST_PORT))
        print(f"  Sent: {msg}")
        time.sleep(1)

    sock.close()

multicast_sender()
```

## Multicast Receiver

```python
import socket
import struct
import json

MCAST_GROUP   = "239.255.0.1"
MCAST_PORT    = 5007
ANY_INTERFACE = "0.0.0.0"

def multicast_receiver():
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)  # Linux

    # Bind to the multicast port on all interfaces
    sock.bind((ANY_INTERFACE, MCAST_PORT))

    # Join the multicast group on all interfaces
    group_req = struct.pack("4sL",
        socket.inet_aton(MCAST_GROUP),
        socket.INADDR_ANY
    )
    sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, group_req)

    print(f"Listening on multicast group {MCAST_GROUP}:{MCAST_PORT}")

    try:
        while True:
            data, addr = sock.recvfrom(4096)
            msg = json.loads(data)
            print(f"From {addr[0]}:{addr[1]} → {msg}")
    except KeyboardInterrupt:
        print("Leaving group...")
    finally:
        # Leave the multicast group
        sock.setsockopt(socket.IPPROTO_IP, socket.IP_DROP_MEMBERSHIP, group_req)
        sock.close()

multicast_receiver()
```

## Multiple Groups

```python
import socket
import struct

def join_groups(sock: socket.socket, groups: list[str]) -> None:
    for group in groups:
        req = struct.pack("4sL",
            socket.inet_aton(group), socket.INADDR_ANY)
        sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, req)
        print(f"Joined {group}")

# Receive from multiple groups on the same socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.bind(("0.0.0.0", 5007))
join_groups(sock, ["239.255.0.1", "239.255.0.2"])
```

## Conclusion

IPv4 multicast sends one packet that is delivered to all group members - more efficient than unicast to each peer. Set `IP_MULTICAST_TTL` to `1` for LAN-only delivery. Call `IP_ADD_MEMBERSHIP` to join a group (sends an IGMP report) and `IP_DROP_MEMBERSHIP` on shutdown. Use addresses in the `239.0.0.0/8` range for private multicast that won't leak onto the internet. Applications for multicast include service discovery, real-time data distribution (market data, game state), and cluster coordination.
