# How to Set Socket Timeouts for IPv4 Connections in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, Sockets, IPv4, Timeout, Networking, Error Handling

Description: Learn how to configure connection and read timeouts for Python IPv4 sockets to prevent applications from hanging indefinitely on slow or unreachable servers.

## Why Timeouts Matter

Without timeouts, a `connect()` or `recv()` call can block forever if the server is unreachable or stops responding. Always set timeouts in production socket code.

## Setting a Timeout with settimeout()

```python
import socket

HOST = "192.168.1.100"
PORT = 9000

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Set 5-second timeout for all socket operations (connect, send, recv)

sock.settimeout(5.0)

try:
    sock.connect((HOST, PORT))
    sock.sendall(b"Hello!")
    response = sock.recv(1024)
    print(response.decode())

except socket.timeout:
    print("Operation timed out")

except ConnectionRefusedError:
    print("Connection refused")

finally:
    sock.close()
```

`settimeout(n)` applies the same timeout to `connect()`, `send()`, and `recv()`.

## Separate Connect and Read Timeouts

To set different timeouts for connection vs. data reading:

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 3-second timeout just for the connection attempt
sock.settimeout(3.0)
try:
    sock.connect(("192.168.1.100", 9000))
except socket.timeout:
    print("Connection timed out after 3 seconds")
    sock.close()
    exit()

# Once connected, switch to a longer read timeout
sock.settimeout(30.0)

sock.sendall(b"heavy request")
try:
    response = sock.recv(65536)
    print(f"Got {len(response)} bytes")
except socket.timeout:
    print("Read timed out after 30 seconds")
finally:
    sock.close()
```

## Using connect_ex() for Non-Blocking Connect with Timeout

```python
import socket
import select

def connect_with_timeout(host: str, port: int, timeout: float) -> socket.socket:
    """Non-blocking connect with custom timeout."""
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setblocking(False)

    result = sock.connect_ex((host, port))
    # EINPROGRESS is expected for non-blocking connect
    if result not in (0, 115):  # 115 = EINPROGRESS
        sock.close()
        raise ConnectionRefusedError(f"connect_ex returned {result}")

    # Wait until the socket becomes writable (connection completed)
    _, writable, _ = select.select([], [sock], [], timeout)
    if not writable:
        sock.close()
        raise TimeoutError(f"Connection to {host}:{port} timed out")

    # Check for connection errors
    err = sock.getsockopt(socket.SOL_SOCKET, socket.SO_ERROR)
    if err:
        sock.close()
        raise ConnectionRefusedError(f"Connection failed: {err}")

    # Switch back to blocking for normal I/O
    sock.setblocking(True)
    return sock


try:
    s = connect_with_timeout("192.168.1.100", 9000, timeout=3.0)
    s.sendall(b"Hello!")
    print(s.recv(1024).decode())
    s.close()
except (TimeoutError, ConnectionRefusedError) as e:
    print(e)
```

## Default Socket Timeout

You can set a global default timeout for all new sockets in the process:

```python
import socket

# All new sockets will have a 10-second timeout by default
socket.setdefaulttimeout(10.0)

# This socket inherits the 10-second timeout
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
```

## Timeout Summary

| Method | Effect |
|--------|--------|
| `settimeout(n)` | n > 0 sets timeout; 0 = non-blocking; None = blocking |
| `setblocking(False)` | Equivalent to `settimeout(0)` |
| `socket.setdefaulttimeout(n)` | Global default for new sockets |

## Conclusion

Always set socket timeouts when connecting to remote servers. Use `settimeout()` for a simple uniform timeout, or switch timeouts between connect and read phases for finer control. For connect-only timeouts in non-blocking mode, `connect_ex()` with `select()` gives maximum flexibility.
