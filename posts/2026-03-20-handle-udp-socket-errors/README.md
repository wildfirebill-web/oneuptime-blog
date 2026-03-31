# How to Handle UDP Socket Errors in Application Code

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, Socket, Error Handling, Python, Linux, Programming

Description: Handle UDP socket errors including ICMP port unreachable delivery, ENOBUFS, ECONNREFUSED, and timeout errors correctly in application code.

## Introduction

UDP error handling is counterintuitive because UDP is connectionless. However, the kernel does deliver some errors to UDP sockets - most notably, ICMP port unreachable messages are delivered back to the sender. Proper error handling means catching these asynchronous errors, handling buffer overflow (`ENOBUFS`), implementing timeouts for `recvfrom()`, and deciding when to retry versus give up.

## ICMP Errors Delivered to UDP Sockets

```python
#!/usr/bin/env python3
# UDP errors from ICMP messages

import socket
import errno

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Send to a port that's not listening (will get ICMP port unreachable back)

# With connected UDP socket, this error is delivered on the NEXT sendto/recvfrom:
sock.connect(('10.20.0.5', 9999))  # Port nothing is listening on

try:
    sock.send(b'hello')
    # ICMP port unreachable comes back asynchronously...
    response = sock.recv(1024)  # This raises ECONNREFUSED
except ConnectionRefusedError as e:
    print(f"ICMP port unreachable received: {e}")
    # The remote port is not open (ICMP type 3, code 3)
except socket.timeout:
    print("No response and no ICMP error (filtered)")

# Note: On UNCONNECTED sockets, ICMP errors are NOT delivered in most cases
# This is a Linux behavior: only "connected" UDP sockets receive ICMP errors
```

## Complete Error Handling

```python
#!/usr/bin/env python3
import socket
import errno
import time

def udp_send_recv(server, port, data, retries=3, timeout=2.0):
    """Send UDP and receive response with full error handling."""
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(timeout)

    for attempt in range(retries):
        try:
            sock.sendto(data, (server, port))
        except OSError as e:
            if e.errno == errno.ENOBUFS:
                # Send buffer is full
                print(f"Buffer full, waiting... (attempt {attempt+1})")
                time.sleep(0.01 * (2 ** attempt))  # Exponential backoff
                continue
            elif e.errno == errno.ENETUNREACH:
                print(f"Network unreachable: {e}")
                break
            elif e.errno == errno.EHOSTUNREACH:
                print(f"Host unreachable: {e}")
                break
            else:
                raise

        try:
            response, addr = sock.recvfrom(65535)
            sock.close()
            return response, addr

        except socket.timeout:
            print(f"Timeout on attempt {attempt+1}/{retries}")
            continue

        except ConnectionRefusedError:
            # ICMP port unreachable (port not open on remote)
            print(f"Connection refused: {server}:{port} has no listener")
            break

        except OSError as e:
            if e.errno == errno.ECONNRESET:
                print(f"Connection reset (ICMP error): {e}")
                break
            else:
                raise

    sock.close()
    return None, None

# Usage:
result, addr = udp_send_recv('10.20.0.5', 5000, b'query data')
if result:
    print(f"Response from {addr}: {result}")
else:
    print("No response received")
```

## Handling ENOBUFS (Send Buffer Full)

```python
import socket
import errno
import time

def send_with_backpressure(sock, data, addr, max_retries=10):
    """Send UDP with retry on ENOBUFS."""
    for i in range(max_retries):
        try:
            sock.sendto(data, addr)
            return True
        except BlockingIOError:
            # Non-blocking socket: buffer full
            time.sleep(0.001 * (i + 1))
        except OSError as e:
            if e.errno == errno.ENOBUFS:
                # Blocking socket: buffer full
                time.sleep(0.001 * (i + 1))
            else:
                raise
    return False  # Failed after all retries
```

## ICMP Error Receipt on Linux

```bash
# By default, ICMP errors only propagate to connected UDP sockets
# Unconnected sockets don't receive ICMP errors (common gotcha)

# Enable ICMP error receipt on unconnected UDP (Linux):
# IP_RECVERR socket option:
python3 -c "
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.setsockopt(socket.IPPROTO_IP, socket.IP_RECVERR, 1)
# Now ICMP errors are queued in error queue
# Read with: sock.recvmsg() and MSG_ERRQUEUE flag
"

# Read error queue (advanced):
import socket
MSG_ERRQUEUE = 8192  # Linux-specific flag
try:
    data, ancdata, flags, addr = sock.recvmsg(1024, 256, MSG_ERRQUEUE)
except BlockingIOError:
    pass  # No error in queue
```

## Conclusion

UDP error handling requires attention to three error classes: ICMP errors delivered asynchronously (`ECONNREFUSED` for port unreachable), send buffer overflow (`ENOBUFS`/`EAGAIN`), and receive timeouts. Use connected UDP sockets to receive ICMP error delivery automatically. Implement exponential backoff retry for `ENOBUFS`. Set receive timeout with `sock.settimeout()` for all `recvfrom()` calls - blocking forever on a UDP socket is always a bug. For unconnected sockets that need ICMP errors, use `IP_RECVERR` with `MSG_ERRQUEUE`.
