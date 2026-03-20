# How to Build an Event-Driven IPv4 Server Using epoll in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, Epoll, Event-Driven, Linux, Networking

Description: Learn how to build a high-performance event-driven IPv4 server using the Linux epoll API directly in Python with the select module, enabling efficient handling of thousands of connections in a...

## What Is epoll?

`epoll` is a Linux kernel I/O event notification mechanism. Unlike `select` (limited to 1024 fds) or `poll` (O(n) scan), `epoll` scales to millions of file descriptors in O(1) time using edge or level triggering.

## Echo Server with select.epoll

```python
import select
import socket

# Note: select.epoll is only available on Linux

EPOLL_EDGE_TRIGGERED = select.EPOLLET  # only notify on transitions
EPOLL_LEVEL_TRIGGERED = 0              # notify while fd is readable

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(("0.0.0.0", 9000))
server.listen(1000)
server.setblocking(False)

epoll  = select.epoll()
epoll.register(server.fileno(), select.EPOLLIN)   # watch for incoming connections

connections = {}   # fd → socket
buffers     = {}   # fd → outbound data

print("epoll echo server on 0.0.0.0:9000")

try:
    while True:
        events = epoll.poll(timeout=-1)   # block until event

        for fd, event in events:
            if fd == server.fileno():
                # New connection
                conn, addr = server.accept()
                conn.setblocking(False)
                connections[conn.fileno()] = conn
                buffers[conn.fileno()]     = b""
                epoll.register(conn.fileno(), select.EPOLLIN)
                print(f"[+] {addr}")

            elif event & select.EPOLLIN:
                # Data available to read
                try:
                    data = connections[fd].recv(4096)
                    if data:
                        buffers[fd] += data
                        # Switch to watching for write events
                        epoll.modify(fd, select.EPOLLOUT)
                    else:
                        # Client closed connection
                        epoll.unregister(fd)
                        connections[fd].close()
                        del connections[fd], buffers[fd]
                except ConnectionResetError:
                    epoll.unregister(fd)
                    connections[fd].close()
                    del connections[fd], buffers[fd]

            elif event & select.EPOLLOUT:
                # Ready to write
                sent = connections[fd].send(buffers[fd])
                buffers[fd] = buffers[fd][sent:]
                if not buffers[fd]:
                    # Nothing more to write - go back to reading
                    epoll.modify(fd, select.EPOLLIN)

            elif event & select.EPOLLHUP:
                # Connection hung up
                epoll.unregister(fd)
                connections[fd].close()
                del connections[fd], buffers[fd]

finally:
    epoll.unregister(server.fileno())
    epoll.close()
    server.close()
```

## epoll vs select vs poll

| Feature | select | poll | epoll |
|---------|--------|------|-------|
| Max fds | 1024 | unlimited | unlimited |
| Scan cost | O(n) | O(n) | O(1) |
| Notification | Level | Level | Level or Edge |
| Platform | POSIX | POSIX | Linux only |

## Performance Tip: Edge-Triggered Mode

```python
# Edge-triggered: epoll only fires when state CHANGES (read → readable)
# You MUST read all available data before returning to epoll.poll()
epoll.register(conn.fileno(), select.EPOLLIN | select.EPOLLET)

# In the handler, read in a loop until EAGAIN:
import errno
while True:
    try:
        chunk = conn.recv(4096)
        if not chunk:
            break  # connection closed
        buffer += chunk
    except BlockingIOError:
        break   # EAGAIN - no more data available right now
```

## Conclusion

`select.epoll` is Linux-only but provides the best performance for servers with thousands of connections. Level-triggered mode (default) fires whenever data is available - simpler to implement. Edge-triggered mode fires only on state changes - requires draining all data in the handler and handling `EAGAIN`. For cross-platform code, use `selectors.DefaultSelector` which uses `epoll` on Linux, `kqueue` on macOS, and falls back to `select` on other platforms. For new Python applications, `asyncio` provides a higher-level API on top of these same OS primitives.
