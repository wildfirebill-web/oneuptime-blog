# How to Disable Nagle's Algorithm for Low-Latency Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Nagle Algorithm, TCP_NODELAY, Low Latency, Linux, Performance

Description: Disable Nagle's algorithm using TCP_NODELAY in various programming languages and server configurations to eliminate 40ms packet coalescing delays for interactive applications.

## Introduction

Disabling Nagle's algorithm (setting TCP_NODELAY) tells the TCP stack to send data immediately rather than waiting to accumulate a full segment. This is the standard configuration for any interactive, real-time, or low-latency application. The trade-off is more small packets on the network, but for applications that send small messages and wait for responses, the latency savings are significant.

## Disabling in Python

```python
import socket

# Method 1: Direct socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)  # Disable Nagle
s.connect(('10.20.0.5', 8080))

# Verify it's set
nodelay = s.getsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY)
print(f"TCP_NODELAY = {nodelay}")  # Should print 1

# Method 2: In a server with accepted connections
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('0.0.0.0', 8080))
server.listen(100)

while True:
    conn, addr = server.accept()
    conn.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)  # Set on each new connection
    handle_connection(conn)
```

## Disabling in Node.js

```javascript
const net = require('net');

// Server: disable Nagle on accepted connections
const server = net.createServer((socket) => {
    socket.setNoDelay(true);  // Equivalent to TCP_NODELAY = 1
    console.log('Client connected, Nagle disabled');

    socket.on('data', (data) => {
        socket.write(data);  // Echo: sends immediately, no coalescing delay
    });
});

server.listen(8080);

// Client: disable Nagle
const client = net.createConnection({ host: '10.20.0.5', port: 8080 }, () => {
    client.setNoDelay(true);
    client.write('Hello');  // Sent immediately
});
```

## Disabling in Go

```go
package main

import (
    "net"
    "fmt"
)

func main() {
    // Dial with TCP_NODELAY
    conn, err := net.Dial("tcp", "10.20.0.5:8080")
    if err != nil {
        panic(err)
    }

    // Type assertion to access SetNoDelay
    tcpConn, ok := conn.(*net.TCPConn)
    if ok {
        tcpConn.SetNoDelay(true)  // Disable Nagle
        fmt.Println("TCP_NODELAY enabled")
    }
}
```

## Disabling in Server Configuration

```nginx
# nginx: disable Nagle for all connections (already default for most responses)
# nginx uses TCP_NOPUSH and TCP_CORK instead of managing Nagle directly
# But for proxy connections:
proxy_socket_keepalive on;
# For upstream connections, nginx handles Nagle automatically
```

```ini
# PostgreSQL server: disable Nagle for client connections
# postgresql.conf:
tcp_keepalives_idle = 60
# PostgreSQL automatically disables Nagle for its protocol connections
```

## Redis and Databases

```bash
# Redis uses TCP_NODELAY by default
# Check redis configuration
redis-cli config get tcp-keepalive

# Verify TCP_NODELAY in use via ss
ss -tin state established 'dport = :6379' | grep nodelay
```

## System-Wide Approach

There is no sysctl to disable Nagle globally for all connections — it must be done per-socket. However, you can patch your application or use LD_PRELOAD to intercept socket calls:

```bash
# For testing only: intercept socket creation with LD_PRELOAD
# This is a debugging technique, not for production
cat > /tmp/nodelay.c << 'EOF'
#include <netinet/tcp.h>
#include <sys/socket.h>

int setsockopt(int fd, int level, int optname, const void *optval, socklen_t optlen) {
    extern int __real_setsockopt(int, int, int, const void *, socklen_t);
    if (level == IPPROTO_TCP && optname == TCP_NODELAY) {
        int enable = 1;
        return __real_setsockopt(fd, level, TCP_NODELAY, &enable, sizeof(enable));
    }
    return __real_setsockopt(fd, level, optname, optval, optlen);
}
EOF
```

## Conclusion

Disabling Nagle's algorithm for interactive applications is straightforward — set TCP_NODELAY on each connected socket after creation. For client code, set it before or right after connect(). For server code, set it on each accepted connection. Every major language's networking library provides a direct method for this. The 40ms latency improvement for request-response protocols is immediate and measurable.
