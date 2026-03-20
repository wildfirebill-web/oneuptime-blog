# How to Handle IPv6 Scope IDs in Socket Programming

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Scope ID, Link-Local, Socket Programming, C, Networking

Description: Understand and use IPv6 scope IDs in socket programming to connect to and bind on link-local addresses that require interface specification.

## Introduction

IPv6 scope IDs are interface identifiers required when using link-local addresses (fe80::/10). Since link-local addresses are not globally unique (every network interface can have the same fe80:: prefix), the scope ID tells the kernel which interface to use. Without a scope ID for link-local addresses, connections will fail with "Invalid argument" or "Network is unreachable."

## When Scope IDs Are Required

| Address Type | Scope ID Required? |
|-------------|-------------------|
| Global unicast (2000::/3) | No (scope_id = 0) |
| Unique local (fd00::/8) | No (scope_id = 0) |
| Link-local (fe80::/10) | **Yes** |
| Loopback (::1) | No (scope_id = 0) |
| Multicast (ff02::/16 link scope) | **Yes** |

## Scope ID Representations

```text
Text format:    fe80::1%eth0         (interface name)
Text format:    fe80::1%2            (interface index)
Numeric:        scope_id = 2         (in struct sockaddr_in6)
```

## Getting Interface Index for Scope ID

```c
#include <net/if.h>
#include <stdio.h>

int main(void) {
    /* Method 1: Convert interface name to index */
    unsigned int idx = if_nametoindex("eth0");
    if (idx == 0) {
        perror("if_nametoindex");
    } else {
        printf("eth0 index: %u\n", idx);
    }

    /* Method 2: Convert interface index to name */
    char ifname[IF_NAMESIZE];
    if (if_indextoname(idx, ifname) != NULL) {
        printf("Interface %u: %s\n", idx, ifname);
    }

    /* List all interfaces and their indices */
    struct if_nameindex *ifaces = if_nameindex();
    if (ifaces) {
        for (struct if_nameindex *i = ifaces; i->if_index != 0; i++) {
            printf("  [%u] %s\n", i->if_index, i->if_name);
        }
        if_freenameindex(ifaces);
    }

    return 0;
}
```

## Connecting to a Link-Local Address

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <net/if.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

int connect_link_local(const char *addr_str, const char *interface, int port) {
    int sockfd = socket(AF_INET6, SOCK_STREAM, 0);
    if (sockfd < 0) { perror("socket"); return -1; }

    struct sockaddr_in6 server;
    memset(&server, 0, sizeof(server));
    server.sin6_family = AF_INET6;
    server.sin6_port   = htons(port);

    /* Parse the IPv6 address (strip zone ID if present in string) */
    const char *clean_addr = addr_str;
    char addr_buf[INET6_ADDRSTRLEN];
    const char *pct = strchr(addr_str, '%');
    if (pct != NULL) {
        /* Address has embedded zone ID, extract just the address part */
        size_t addr_len = pct - addr_str;
        strncpy(addr_buf, addr_str, addr_len);
        addr_buf[addr_len] = '\0';
        clean_addr = addr_buf;
        /* Use the interface from the zone ID if none provided */
        if (interface == NULL) {
            interface = pct + 1;
        }
    }

    if (inet_pton(AF_INET6, clean_addr, &server.sin6_addr) != 1) {
        fprintf(stderr, "Invalid IPv6 address: %s\n", clean_addr);
        close(sockfd);
        return -1;
    }

    /* Set scope ID from interface name */
    if (interface != NULL) {
        server.sin6_scope_id = if_nametoindex(interface);
        if (server.sin6_scope_id == 0) {
            fprintf(stderr, "Interface not found: %s\n", interface);
            close(sockfd);
            return -1;
        }
    }

    printf("Connecting to [%s%%%s]:%d (scope_id=%u)\n",
           clean_addr, interface ? interface : "none",
           port, server.sin6_scope_id);

    if (connect(sockfd, (struct sockaddr *)&server, sizeof(server)) < 0) {
        perror("connect");
        close(sockfd);
        return -1;
    }

    return sockfd;
}

int main(void) {
    /* Connect to a link-local address on eth0 */
    int fd = connect_link_local("fe80::1", "eth0", 8080);
    if (fd >= 0) {
        printf("Connected!\n");
        close(fd);
    }

    /* With zone ID embedded in address string */
    fd = connect_link_local("fe80::1%eth0", NULL, 8080);
    if (fd >= 0) {
        printf("Connected via zone ID!\n");
        close(fd);
    }

    return 0;
}
```

## Binding to a Link-Local Address

```c
int bind_link_local_server(const char *addr_str, const char *interface, int port) {
    int sockfd = socket(AF_INET6, SOCK_STREAM, 0);

    struct sockaddr_in6 addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin6_family   = AF_INET6;
    addr.sin6_port     = htons(port);
    addr.sin6_scope_id = if_nametoindex(interface);  /* Required for link-local */

    inet_pton(AF_INET6, addr_str, &addr.sin6_addr);

    int reuse = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    if (bind(sockfd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        perror("bind");
        close(sockfd);
        return -1;
    }
    listen(sockfd, 5);
    printf("Server bound to [%s%%%s]:%d\n", addr_str, interface, port);
    return sockfd;
}
```

## Scope IDs in Python

```python
import socket

# Connect to link-local with scope ID using %interface notation

def connect_link_local(addr: str, interface: str, port: int):
    # Get interface index
    scope_id = socket.if_nametoindex(interface)

    sock = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
    sock.settimeout(10)

    # 4-tuple for IPv6: (host, port, flowinfo, scope_id)
    sock.connect((addr, port, 0, scope_id))
    return sock

# Usage
try:
    sock = connect_link_local('fe80::1', 'eth0', 8080)
    print(f"Connected to {sock.getpeername()}")
    sock.close()
except OSError as e:
    print(f"Failed: {e}")
```

## Conclusion

IPv6 scope IDs are essential for link-local address communication. Use `if_nametoindex()` to convert interface names to scope ID integers, then set `sockaddr_in6.sin6_scope_id` before calling `connect()` or `bind()`. In string notation, zone IDs appear after `%` (e.g., `fe80::1%eth0`). For link-local multicast, the same scope ID requirement applies.
