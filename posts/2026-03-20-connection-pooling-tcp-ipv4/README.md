# How to Implement Connection Pooling for IPv4 TCP Clients

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, TCP, Connection Pool, Networking, Performance

Description: Learn how to implement TCP connection pooling for IPv4 clients in Python to reuse connections, reduce handshake overhead, and improve throughput for services that accept persistent connections.

## Why Connection Pooling?

Each new TCP connection requires a 3-way handshake (SYN → SYN-ACK → ACK), which adds latency. With TLS, the handshake is even more expensive. Connection pooling reuses established connections for multiple requests, eliminating this overhead.

## Simple Thread-Safe Connection Pool

```python
import socket
import threading
import queue
import contextlib
import time

class TCPConnectionPool:
    def __init__(self, host: str, port: int, size: int = 10,
                 timeout: float = 5.0, idle_ttl: float = 60.0):
        self.host     = host
        self.port     = port
        self.timeout  = timeout
        self.idle_ttl = idle_ttl
        self._pool: queue.Queue = queue.Queue(maxsize=size)
        self._lock  = threading.Lock()
        self._size  = size

        # Pre-create connections
        for _ in range(size):
            self._pool.put(self._create())

    def _create(self) -> socket.socket:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(self.timeout)
        s.connect((self.host, self.port))
        return s

    def _is_alive(self, sock: socket.socket) -> bool:
        """Non-blocking check if connection is still open."""
        try:
            sock.setblocking(False)
            data = sock.recv(1, socket.MSG_PEEK)
            sock.setblocking(True)
            sock.settimeout(self.timeout)
            return len(data) > 0
        except BlockingIOError:
            sock.setblocking(True)
            sock.settimeout(self.timeout)
            return True  # no data but still connected
        except Exception:
            return False

    @contextlib.contextmanager
    def get(self):
        """Context manager that yields a connection and returns it to pool."""
        conn = None
        try:
            conn = self._pool.get(timeout=self.timeout)
            # Check if connection is still healthy
            if not self._is_alive(conn):
                try: conn.close()
                except Exception: pass
                conn = self._create()
            yield conn
        except queue.Empty:
            # Pool exhausted — create a temporary connection
            conn = self._create()
            yield conn
            try: conn.close()
            except Exception: pass
            conn = None  # don't return to pool
        finally:
            if conn is not None:
                try:
                    self._pool.put_nowait(conn)
                except queue.Full:
                    try: conn.close()
                    except Exception: pass

pool = TCPConnectionPool("192.168.1.10", 9000, size=10)

def send_request(payload: bytes) -> bytes:
    with pool.get() as conn:
        conn.sendall(payload)
        return conn.recv(4096)

# Use from multiple threads
import concurrent.futures
with concurrent.futures.ThreadPoolExecutor(max_workers=20) as executor:
    futures = [executor.submit(send_request, b"ping") for _ in range(100)]
    results = [f.result() for f in futures]
```

## asyncio Connection Pool

```python
import asyncio
from contextlib import asynccontextmanager

class AsyncTCPPool:
    def __init__(self, host: str, port: int, size: int = 10):
        self.host = host
        self.port = port
        self._queue: asyncio.Queue = asyncio.Queue(maxsize=size)
        self._size  = size

    async def start(self) -> None:
        for _ in range(self._size):
            r, w = await asyncio.open_connection(self.host, self.port)
            await self._queue.put((r, w))

    @asynccontextmanager
    async def get(self):
        reader, writer = await self._queue.get()
        try:
            yield reader, writer
        finally:
            if not writer.is_closing():
                await self._queue.put((reader, writer))
            else:
                r, w = await asyncio.open_connection(self.host, self.port)
                await self._queue.put((r, w))

async def main():
    pool = AsyncTCPPool("192.168.1.10", 9000, size=5)
    await pool.start()

    async with pool.get() as (reader, writer):
        writer.write(b"hello\n")
        await writer.drain()
        reply = await reader.readline()
        print(f"Reply: {reply.decode().strip()}")

asyncio.run(main())
```

## Conclusion

A connection pool maintains a fixed set of open TCP connections. The `contextlib.contextmanager` pattern ensures connections are always returned to the pool even if an exception occurs. Check connection health before use with a non-blocking `MSG_PEEK` recv — create a fresh connection if the old one has been closed by the server. Size the pool to the expected concurrent request count — too small causes queuing, too large wastes server resources. For asyncio applications, use an `asyncio.Queue` of (reader, writer) pairs as the pool backing store.
