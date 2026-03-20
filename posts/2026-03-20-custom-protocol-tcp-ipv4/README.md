# How to Implement a Custom Protocol over IPv4 TCP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, TCP, Custom Protocol, Networking, Framing

Description: Learn how to design and implement a custom application protocol over IPv4 TCP in Python, covering message framing, versioning, command dispatch, and error handling.

## Protocol Design Principles

A custom TCP protocol needs:
1. **Framing** — how to find message boundaries in the TCP byte stream
2. **Header** — version, message type, payload length, optional checksum
3. **Command dispatch** — route messages to handlers
4. **Error handling** — unknown commands, malformed messages

## Wire Format

```
+------+------+--------+--------+---------+
| VER  | CMD  | FLAGS  | LENGTH |  PAYLOAD|
| 1B   | 1B   | 2B     | 4B     |  N bytes|
+------+------+--------+--------+---------+
  Total header = 8 bytes
```

## Protocol Implementation

```python
import struct
import socket
import json
import enum

HEADER_SIZE = 8
MAGIC       = 0x01   # version byte

class Cmd(enum.IntEnum):
    PING    = 1
    PONG    = 2
    ECHO    = 3
    ERROR   = 255

class Message:
    def __init__(self, cmd: Cmd, payload: bytes = b"", flags: int = 0):
        self.cmd     = cmd
        self.payload = payload
        self.flags   = flags

    def pack(self) -> bytes:
        header = struct.pack("!BBHI",
            MAGIC, int(self.cmd), self.flags, len(self.payload))
        return header + self.payload

    @classmethod
    def unpack_header(cls, data: bytes) -> tuple[int, int, int, int]:
        """Returns (version, cmd, flags, payload_len)."""
        if len(data) < HEADER_SIZE:
            raise ValueError("Incomplete header")
        return struct.unpack("!BBHI", data[:HEADER_SIZE])

def recvn(sock: socket.socket, n: int) -> bytes:
    buf = b""
    while len(buf) < n:
        chunk = sock.recv(n - len(buf))
        if not chunk:
            raise ConnectionError("Connection closed")
        buf += chunk
    return buf

def recv_message(sock: socket.socket) -> Message:
    raw_header = recvn(sock, HEADER_SIZE)
    ver, cmd, flags, length = Message.unpack_header(raw_header)
    if ver != MAGIC:
        raise ValueError(f"Unknown protocol version: {ver:#x}")
    payload = recvn(sock, length) if length else b""
    return Message(Cmd(cmd), payload, flags)

def send_message(sock: socket.socket, msg: Message) -> None:
    sock.sendall(msg.pack())
```

## Protocol Server

```python
import socket
import threading

HANDLERS = {}

def command(cmd: Cmd):
    def dec(f):
        HANDLERS[cmd] = f
        return f
    return dec

@command(Cmd.PING)
def handle_ping(msg: Message) -> Message:
    return Message(Cmd.PONG, b"")

@command(Cmd.ECHO)
def handle_echo(msg: Message) -> Message:
    return Message(Cmd.ECHO, msg.payload)

def serve_client(conn: socket.socket) -> None:
    with conn:
        while True:
            try:
                msg  = recv_message(conn)
                handler = HANDLERS.get(msg.cmd)
                if handler:
                    reply = handler(msg)
                else:
                    reply = Message(Cmd.ERROR,
                                    f"Unknown command: {msg.cmd}".encode())
                send_message(conn, reply)
            except (ConnectionError, struct.error):
                break

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(("0.0.0.0", 9000))
server.listen(50)
print("Custom protocol server on 0.0.0.0:9000")

while True:
    conn, addr = server.accept()
    threading.Thread(target=serve_client, args=(conn,), daemon=True).start()
```

## Protocol Client

```python
import socket

def make_client(host: str, port: int) -> socket.socket:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host, port))
    return s

with make_client("192.168.1.10", 9000) as s:
    # Send PING
    send_message(s, Message(Cmd.PING))
    resp = recv_message(s)
    print(f"PING → {resp.cmd.name}")  # PONG

    # Send ECHO with JSON payload
    payload = json.dumps({"hello": "world"}).encode()
    send_message(s, Message(Cmd.ECHO, payload))
    resp = recv_message(s)
    print(f"ECHO → {json.loads(resp.payload)}")
```

## Conclusion

A custom TCP protocol needs a fixed-size header with a version byte (for future compatibility), a command/message-type byte, and a payload length field. The `recvn` helper ensures TCP fragmentation is handled correctly. Use a `cmd → handler` dispatch table to keep the server loop clean. Always validate the protocol version to reject incompatible clients early. When adding new commands, increment the version only when backward compatibility breaks — otherwise just add new command codes.
