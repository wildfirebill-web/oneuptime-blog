# How to Create a UDP Server and Client in Python with IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, UDP, Sockets, IPv4, Networking, Datagram

Description: Learn how to create a UDP server and client in Python using IPv4 with the socket module, including handling datagrams and sender addresses.

## UDP vs TCP

UDP (User Datagram Protocol) is connectionless—there's no handshake, no guaranteed delivery, and no ordering. This makes it faster and lower-overhead than TCP, ideal for DNS, video streaming, VoIP, and real-time telemetry where occasional packet loss is acceptable.

## UDP Server

```python
import socket

HOST = "0.0.0.0"   # Listen on all IPv4 interfaces
PORT = 9001        # UDP port to bind

# AF_INET = IPv4, SOCK_DGRAM = UDP
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as server:
    server.bind((HOST, PORT))
    print(f"UDP server listening on {HOST}:{PORT}")

    while True:
        # recvfrom returns (data, (sender_ip, sender_port))
        data, addr = server.recvfrom(4096)
        print(f"Received {len(data)} bytes from {addr}: {data.decode('utf-8')}")

        # Send a response back to the sender
        reply = f"Echo: {data.decode('utf-8')}".encode("utf-8")
        server.sendto(reply, addr)
```

## UDP Client

```python
import socket

SERVER_HOST = "127.0.0.1"
SERVER_PORT = 9001

# UDP sockets do not need an explicit connect()
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as client:
    message = "Hello, UDP server!"

    # sendto sends a datagram directly to the target address
    client.sendto(message.encode("utf-8"), (SERVER_HOST, SERVER_PORT))

    # Set a timeout so we don't wait forever for a response
    client.settimeout(3.0)

    try:
        response, server_addr = client.recvfrom(4096)
        print(f"Response from {server_addr}: {response.decode('utf-8')}")
    except socket.timeout:
        print("No response received within timeout period")
```

## Using connect() on a UDP Socket

You can call `connect()` on a UDP socket to set a default remote address, enabling use of `send()` and `recv()` instead of `sendto()` and `recvfrom()`:

```python
import socket

with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as client:
    # "Connect" sets default destination; no actual handshake occurs
    client.connect(("127.0.0.1", 9001))

    # Now use send/recv instead of sendto/recvfrom
    client.send(b"Hello!")
    client.settimeout(3.0)
    try:
        response = client.recv(4096)
        print(response.decode())
    except socket.timeout:
        print("No response")
```

## Handling Multiple Clients

UDP servers naturally handle multiple clients without threads because each `recvfrom()` call returns the sender's address:

```python
import socket

with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as server:
    server.bind(("0.0.0.0", 9001))
    print("UDP echo server running")

    while True:
        data, addr = server.recvfrom(65535)   # Max UDP datagram size

        # Log which client sent the message
        print(f"[{addr[0]}:{addr[1]}] {data.decode(errors='replace')}")

        # Reply to the specific client
        server.sendto(data, addr)
```

## Key UDP Limitations

- No guaranteed delivery — packets may be lost
- No ordering guarantee — packets may arrive out of order
- No flow control — receiver can be overwhelmed
- Max datagram size: 65507 bytes for IPv4 UDP

## Conclusion

UDP servers use `bind()` and `recvfrom()`, while clients use `sendto()` directly. There is no connection lifecycle to manage, making UDP simpler but requiring your application protocol to handle reliability if needed. UDP is the right choice for low-latency, loss-tolerant applications like DNS queries, game state updates, or metrics collection.
