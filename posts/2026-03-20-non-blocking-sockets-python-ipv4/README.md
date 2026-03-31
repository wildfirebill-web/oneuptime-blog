# How to Implement Non-Blocking Sockets in Python for IPv4 Communication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, Non-Blocking, Socket, IPv4, Networking, SELECT

Description: Learn how to implement non-blocking IPv4 sockets in Python to handle multiple connections in a single thread without blocking on I/O operations.

## Blocking vs Non-Blocking Sockets

By default, Python socket calls like `recv()` and `accept()` block-the thread sleeps until data arrives or a connection is made. Non-blocking sockets raise `BlockingIOError` (or `socket.error` with errno `EAGAIN`) immediately if the operation can't complete, allowing a single thread to manage multiple connections.

## Setting a Socket to Non-Blocking Mode

```python
import socket

# Method 1: setblocking(False)

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setblocking(False)

# Method 2: equivalent settimeout(0)
sock.settimeout(0)
```

## Non-Blocking Server with select()

The `select` module multiplexes multiple sockets, blocking only until at least one is ready for I/O:

```python
import socket
import select

HOST = "0.0.0.0"
PORT = 9003

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind((HOST, PORT))
server.listen(50)
server.setblocking(False)

# Track all sockets we're monitoring
inputs = [server]     # Sockets to watch for readability
outputs = []          # Sockets to watch for writability
message_queues: dict[socket.socket, list[bytes]] = {}

print(f"Non-blocking server on {HOST}:{PORT}")

while inputs:
    # select() returns readable, writable, exceptional sockets
    # Timeout of 1 second prevents indefinite blocking
    readable, writable, exceptional = select.select(inputs, outputs, inputs, 1.0)

    for sock in readable:
        if sock is server:
            # New incoming connection
            conn, addr = server.accept()
            print(f"New connection from {addr}")
            conn.setblocking(False)
            inputs.append(conn)
            message_queues[conn] = []
        else:
            # Existing client has data
            try:
                data = sock.recv(4096)
                if data:
                    # Queue response to be sent when socket is writable
                    message_queues[sock].append(data)
                    if sock not in outputs:
                        outputs.append(sock)
                else:
                    # Empty data = client disconnected
                    print(f"Client disconnected")
                    inputs.remove(sock)
                    if sock in outputs:
                        outputs.remove(sock)
                    del message_queues[sock]
                    sock.close()
            except ConnectionResetError:
                inputs.remove(sock)
                sock.close()

    for sock in writable:
        if message_queues.get(sock):
            # Send the next queued message
            msg = message_queues[sock].pop(0)
            sock.send(msg)
        else:
            # Nothing to send; stop watching for writability
            outputs.remove(sock)

    for sock in exceptional:
        print(f"Exception on socket {sock.getpeername()}")
        inputs.remove(sock)
        if sock in outputs:
            outputs.remove(sock)
        del message_queues[sock]
        sock.close()
```

## Handling EAGAIN in Non-Blocking Reads

When reading without `select`, handle the `BlockingIOError`:

```python
import socket
import errno

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setblocking(False)
sock.connect_ex(("127.0.0.1", 9003))  # connect_ex doesn't raise on EINPROGRESS

try:
    data = sock.recv(1024)
except BlockingIOError as e:
    if e.errno == errno.EAGAIN:
        # No data available right now; try again later
        pass
```

## When to Use Non-Blocking Sockets

| Approach | Best For |
|----------|---------|
| Blocking + threads | Simple multi-client servers |
| Non-blocking + select | Moderate concurrency, single thread |
| asyncio | High concurrency, modern Python code |

## Conclusion

Non-blocking sockets combined with `select()` allow a single Python thread to manage hundreds of connections. The trade-off is more complex application code compared to blocking sockets with threads. For new projects, `asyncio` provides a higher-level abstraction over the same non-blocking I/O primitives.
