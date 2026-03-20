# How to Connect to IPv6 Link-Local Addresses in Code

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Link-Local, Socket Programming, Networking, C, Python, Go

Description: Connect to IPv6 link-local addresses (fe80::/10) in application code by specifying the required interface scope ID across C, Python, and Go.

## Introduction

IPv6 link-local addresses (fe80::/10) are automatically assigned to every IPv6-capable interface but are not globally routable. Connecting to a link-local address requires specifying a scope ID (the outgoing interface) because the same fe80:: prefix exists on every interface. Without the scope ID, the kernel cannot determine which interface to use.

## Identifying Link-Local Addresses

```bash
# List all link-local addresses on your system

ip -6 addr show | grep "scope link"

# Output example:
# inet6 fe80::1/64 scope link
#    valid_lft forever preferred_lft forever

# Get just the addresses
ip -6 addr show scope link | grep inet6 | awk '{print $2}'
```

## Connecting in C

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <net/if.h>
#include <string.h>
#include <stdio.h>
#include <unistd.h>

/*
 * Connect to a link-local IPv6 address.
 *
 * addr_str:  The link-local address, e.g. "fe80::1" or "fe80::1%eth0"
 * interface: The interface name, e.g. "eth0" (can be NULL if in addr_str)
 * port:      The TCP port to connect to
 */
int connect_link_local(const char *addr_str, const char *interface, int port) {
    char clean_addr[INET6_ADDRSTRLEN];
    const char *iface = interface;

    /* Handle "fe80::1%eth0" embedded zone ID format */
    const char *pct = strchr(addr_str, '%');
    if (pct != NULL) {
        size_t addr_len = pct - addr_str;
        strncpy(clean_addr, addr_str, addr_len);
        clean_addr[addr_len] = '\0';
        if (iface == NULL) iface = pct + 1;
    } else {
        strncpy(clean_addr, addr_str, sizeof(clean_addr)-1);
    }

    if (iface == NULL) {
        fprintf(stderr, "Interface required for link-local address\n");
        return -1;
    }

    struct sockaddr_in6 server;
    memset(&server, 0, sizeof(server));
    server.sin6_family   = AF_INET6;
    server.sin6_port     = htons(port);
    server.sin6_scope_id = if_nametoindex(iface);  /* ← Required for link-local */

    if (server.sin6_scope_id == 0) {
        fprintf(stderr, "Interface not found: %s\n", iface);
        return -1;
    }

    if (inet_pton(AF_INET6, clean_addr, &server.sin6_addr) != 1) {
        fprintf(stderr, "Invalid address: %s\n", clean_addr);
        return -1;
    }

    int sockfd = socket(AF_INET6, SOCK_STREAM, 0);
    if (connect(sockfd, (struct sockaddr *)&server, sizeof(server)) < 0) {
        perror("connect");
        close(sockfd);
        return -1;
    }

    printf("Connected to [%s%%%s]:%d\n", clean_addr, iface, port);
    return sockfd;
}

int main(void) {
    /* Connect to neighbor's link-local address on eth0 */
    int fd = connect_link_local("fe80::1", "eth0", 8080);
    if (fd >= 0) {
        char buf[256];
        read(fd, buf, sizeof(buf)-1);
        printf("Server replied\n");
        close(fd);
    }
    return 0;
}
```

## Connecting in Python

```python
import socket

def connect_link_local(addr: str, interface: str, port: int) -> socket.socket:
    """
    Connect to a link-local IPv6 address.

    Args:
        addr: Link-local address like "fe80::1"
        interface: Interface name like "eth0"
        port: TCP port number
    """
    # Get the interface index (needed as scope_id)
    scope_id = socket.if_nametoindex(interface)

    sock = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
    sock.settimeout(10)

    # IPv6 connect address is a 4-tuple: (host, port, flowinfo, scope_id)
    sock.connect((addr, port, 0, scope_id))
    return sock

# Also handle embedded zone ID: "fe80::1%eth0"
def parse_link_local(addr_with_zone: str) -> tuple[str, str | None]:
    """Parse 'fe80::1%eth0' into ('fe80::1', 'eth0')."""
    if '%' in addr_with_zone:
        addr, iface = addr_with_zone.split('%', 1)
        return addr, iface
    return addr_with_zone, None

# Usage examples
try:
    # Direct
    sock = connect_link_local('fe80::1', 'eth0', 8080)
    print(f"Connected to {sock.getpeername()}")
    sock.close()
except OSError as e:
    print(f"Error: {e}")

# Auto-discover link-local neighbors and print them
def list_link_local_addresses():
    import psutil
    results = []
    for iface, addrs in psutil.net_if_addrs().items():
        for addr in addrs:
            if addr.family == socket.AF_INET6 and addr.address.startswith('fe80'):
                results.append((addr.address, iface))
    return results

for addr, iface in list_link_local_addresses():
    print(f"Link-local: {addr} on {iface}")
```

## Connecting in Go

```go
package main

import (
    "fmt"
    "net"
    "time"
)

func connectLinkLocal(addr, iface, port string) (net.Conn, error) {
    // Go uses "fe80::1%eth0" zone ID notation in address strings
    target := fmt.Sprintf("[%s%%%s]:%s", addr, iface, port)

    return net.DialTimeout("tcp6", target, 10*time.Second)
}

func main() {
    conn, err := connectLinkLocal("fe80::1", "eth0", "8080")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer conn.Close()
    fmt.Printf("Connected to %s\n", conn.RemoteAddr())
}
```

## Testing Link-Local Connections

```bash
# Start a server on a link-local address
# (Use nc or python to listen)
nc -6 -l -p 8080 &

# Connect to it using nc with scope ID
nc -6 fe80::1%eth0 8080

# Or use the numeric interface index
SCOPE=$(cat /sys/class/net/eth0/ifindex)
nc -6 "fe80::1%$SCOPE" 8080

# Test with curl
curl -6 --interface eth0 "http://[fe80::1%25eth0]:8080/"
# Note: %25 is URL-encoded %
```

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Network is unreachable` | Missing scope ID | Add `if_nametoindex()` result to `sin6_scope_id` |
| `Invalid argument` | Scope ID is 0 | Check interface name exists with `ip link show` |
| `Connection refused` | Server not listening | Verify server binds to link-local on same interface |
| `No route to host` | Different interface | Ensure client and server are on same link |

## Conclusion

Connecting to IPv6 link-local addresses requires specifying the outgoing interface as a scope ID in the socket address structure. In C, set `sin6_scope_id = if_nametoindex("eth0")`. In Python, pass the scope ID as the 4th element of the address 4-tuple. In Go, append `%interface` to the address string. Without the scope ID, the kernel cannot route link-local packets.
