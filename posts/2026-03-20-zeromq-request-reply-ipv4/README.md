# How to Build a Request-Reply Pattern over IPv4 Using ZeroMQ

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ZeroMQ, IPv4, Request-Reply, Python, Messaging, Networking

Description: Learn how to implement the ZeroMQ request-reply (REQ/REP) messaging pattern over IPv4, with Python examples for synchronous and asynchronous request-reply services.

## Installation

```bash
pip install pyzmq
```

## Basic REQ/REP Pattern

```python
# --- server.py (REP socket — receives request, sends reply) ---
import zmq

ctx    = zmq.Context()
socket = ctx.socket(zmq.REP)
socket.bind("tcp://0.0.0.0:5555")   # bind to all IPv4 interfaces
print("ZeroMQ REP server on tcp://0.0.0.0:5555")

while True:
    message = socket.recv_string()
    print(f"Request: {message}")
    socket.send_string(f"Reply to: {message}")


# --- client.py (REQ socket — sends request, receives reply) ---
import zmq

ctx    = zmq.Context()
socket = ctx.socket(zmq.REQ)
socket.connect("tcp://192.168.1.10:5555")

for i in range(5):
    socket.send_string(f"Request #{i}")
    reply = socket.recv_string()
    print(f"Server said: {reply}")

socket.close()
ctx.term()
```

## JSON Request-Reply Service

```python
import zmq
import json
import threading

def service(bind_addr: str = "tcp://0.0.0.0:5555") -> None:
    ctx    = zmq.Context()
    socket = ctx.socket(zmq.REP)
    socket.bind(bind_addr)
    print(f"JSON service on {bind_addr}")

    while True:
        raw = socket.recv_bytes()
        try:
            req = json.loads(raw)
            action = req.get("action")

            if action == "echo":
                resp = {"status": "ok", "data": req.get("data")}
            elif action == "ping":
                resp = {"status": "ok", "pong": True}
            else:
                resp = {"status": "error", "message": f"unknown action: {action}"}
        except json.JSONDecodeError:
            resp = {"status": "error", "message": "invalid JSON"}

        socket.send_bytes(json.dumps(resp).encode())

# Client
def call(server_addr: str, action: str, **kwargs) -> dict:
    ctx    = zmq.Context.instance()
    socket = ctx.socket(zmq.REQ)
    socket.setsockopt(zmq.RCVTIMEO, 3000)   # 3-second timeout
    socket.connect(server_addr)
    try:
        socket.send_bytes(json.dumps({"action": action, **kwargs}).encode())
        raw = socket.recv_bytes()
        return json.loads(raw)
    except zmq.Again:
        return {"status": "error", "message": "timeout"}
    finally:
        socket.close()

# Usage
result = call("tcp://192.168.1.10:5555", "echo", data="hello")
print(result)  # {"status": "ok", "data": "hello"}
```

## Async with asyncio + zmq.asyncio

```python
import asyncio
import zmq.asyncio

async def server(bind_addr: str = "tcp://0.0.0.0:5555") -> None:
    ctx    = zmq.asyncio.Context()
    socket = ctx.socket(zmq.REP)
    socket.bind(bind_addr)
    print(f"Async ZMQ server on {bind_addr}")

    while True:
        msg = await socket.recv_string()
        await asyncio.sleep(0)  # yield to event loop
        await socket.send_string(f"async reply: {msg}")

asyncio.run(server())
```

## REQ/REP vs REST

| Aspect | ZeroMQ REQ/REP | REST (HTTP) |
|--------|---------------|-------------|
| Protocol | Custom over TCP | HTTP/1.1 or HTTP/2 |
| Overhead | Very low | Header overhead |
| Discovery | Manual address config | URL routing |
| Patterns | REQ/REP, PUB/SUB, PUSH/PULL | Request/Response |
| Use Case | Internal microservices | Public APIs |

## Conclusion

ZeroMQ's `REQ`/`REP` sockets implement the request-reply pattern with zero application-level framing code — messages are delivered atomically. Bind the `REP` socket to `tcp://0.0.0.0:port` to accept on all IPv4 interfaces. The `REQ` socket's strict alternating send/recv discipline means you must always receive a reply before sending the next request — for concurrent requests use `DEALER`/`ROUTER` sockets instead. Set `RCVTIMEO` on client sockets to avoid blocking forever when the server is unavailable.
