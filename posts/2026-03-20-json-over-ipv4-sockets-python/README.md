# How to Send and Receive JSON Data over IPv4 Sockets in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, JSON, TCP, Sockets, IPv4, Networking, Protocol

Description: Learn how to send and receive structured JSON data over IPv4 TCP sockets in Python using length-prefixed framing to handle message boundaries.

## The Framing Problem

TCP is a byte stream with no message boundaries. Sending `json.dumps(data).encode()` may arrive in multiple chunks, or multiple JSON messages may arrive in a single `recv()`. You must add framing—a length prefix—to delineate message boundaries.

## JSON-over-TCP Protocol

We use a 4-byte big-endian length prefix followed by the UTF-8 encoded JSON payload:

```
[4-byte length][JSON payload bytes]
```

## Helper Functions

```python
import json
import socket
import struct


def send_json(sock: socket.socket, data: dict) -> None:
    """Serialize data to JSON and send with a 4-byte length prefix."""
    payload = json.dumps(data).encode("utf-8")
    # Pack the length as a 4-byte big-endian unsigned integer
    header = struct.pack(">I", len(payload))
    sock.sendall(header + payload)


def recv_json(sock: socket.socket) -> dict:
    """Receive a length-prefixed JSON message from the socket."""
    # Read exactly 4 bytes for the length header
    raw_len = recvn(sock, 4)
    if not raw_len:
        raise ConnectionError("Connection closed while reading header")

    msg_len = struct.unpack(">I", raw_len)[0]

    # Read exactly msg_len bytes for the payload
    raw_payload = recvn(sock, msg_len)
    if not raw_payload:
        raise ConnectionError("Connection closed while reading payload")

    return json.loads(raw_payload.decode("utf-8"))


def recvn(sock: socket.socket, n: int) -> bytes:
    """Read exactly n bytes from the socket."""
    buf = b""
    while len(buf) < n:
        chunk = sock.recv(n - len(buf))
        if not chunk:
            return b""
        buf += chunk
    return buf
```

## JSON TCP Server

```python
import socket
import threading

HOST = "0.0.0.0"
PORT = 9009


def handle_client(conn: socket.socket, addr: tuple) -> None:
    print(f"Client connected: {addr}")
    with conn:
        while True:
            try:
                msg = recv_json(conn)
                print(f"Received from {addr}: {msg}")

                # Echo the message back with a status field
                response = {"status": "ok", "echo": msg}
                send_json(conn, response)

            except (ConnectionError, json.JSONDecodeError) as e:
                print(f"Client {addr} error: {e}")
                break


with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind((HOST, PORT))
    srv.listen(10)
    print(f"JSON server on {PORT}")

    while True:
        conn, addr = srv.accept()
        threading.Thread(target=handle_client, args=(conn, addr), daemon=True).start()
```

## JSON TCP Client

```python
import socket

SERVER_HOST = "127.0.0.1"
SERVER_PORT = 9009

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as client:
    client.connect((SERVER_HOST, SERVER_PORT))

    # Send a JSON request
    request = {
        "action": "create_user",
        "username": "alice",
        "email": "alice@example.com"
    }
    send_json(client, request)

    # Receive and print the JSON response
    response = recv_json(client)
    print(f"Server response: {response}")
```

## Handling Large JSON Payloads

For large objects (e.g., bulk data transfers), consider compression:

```python
import zlib

def send_json_compressed(sock: socket.socket, data: dict) -> None:
    """Send JSON with zlib compression."""
    payload = zlib.compress(json.dumps(data).encode("utf-8"))
    header = struct.pack(">I", len(payload))
    sock.sendall(header + payload)

def recv_json_compressed(sock: socket.socket) -> dict:
    raw_len = recvn(sock, 4)
    msg_len = struct.unpack(">I", raw_len)[0]
    raw_payload = recvn(sock, msg_len)
    return json.loads(zlib.decompress(raw_payload).decode("utf-8"))
```

## Conclusion

Sending JSON over TCP requires a framing layer. The 4-byte length prefix pattern is simple, efficient, and widely used. The `recvn()` helper ensures you always read exactly the expected number of bytes, handling TCP's natural stream segmentation transparently.
