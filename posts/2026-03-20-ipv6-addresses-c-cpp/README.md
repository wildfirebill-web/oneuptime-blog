# How to Handle IPv6 Addresses in C/C++ Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: C, C++, IPv6, Socket Programming, POSIX, Networking, Systems Programming

Description: Handle, parse, and use IPv6 addresses in C and C++ applications using POSIX socket APIs including inet_pton, inet_ntop, and AF_INET6 socket structures.

## Introduction

C and C++ applications use POSIX socket APIs for IPv6 networking. The key functions are `inet_pton()` for parsing text to binary, `inet_ntop()` for binary to text, and the `sockaddr_in6` structure for address storage. This guide covers the essential patterns for IPv6-capable C/C++ code.

## Key IPv6 Data Structures

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>

/* IPv6 socket address structure */
struct sockaddr_in6 {
    sa_family_t     sin6_family;    /* AF_INET6 */
    in_port_t       sin6_port;      /* Port number (network byte order) */
    uint32_t        sin6_flowinfo;  /* IPv6 flow info (usually 0) */
    struct in6_addr sin6_addr;      /* IPv6 address (128 bits) */
    uint32_t        sin6_scope_id;  /* Scope ID for link-local addresses */
};

/* The 128-bit IPv6 address */
struct in6_addr {
    uint8_t s6_addr[16];  /* 16 bytes = 128 bits */
};
```

## Parsing IPv6 Addresses with inet_pton

```c
#include <stdio.h>
#include <string.h>
#include <arpa/inet.h>
#include <netinet/in.h>

int parse_ipv6(const char *addr_str, struct in6_addr *result) {
    /* inet_pton returns 1 on success, 0 if invalid, -1 on error */
    int ret = inet_pton(AF_INET6, addr_str, result);
    if (ret != 1) {
        fprintf(stderr, "Invalid IPv6 address: %s\n", addr_str);
        return -1;
    }
    return 0;
}

int main(void) {
    struct in6_addr addr;
    const char *test = "2001:db8::1";

    if (parse_ipv6(test, &addr) == 0) {
        /* Convert back to string with inet_ntop */
        char buf[INET6_ADDRSTRLEN];  /* 46 bytes, max IPv6 string length */
        inet_ntop(AF_INET6, &addr, buf, sizeof(buf));
        printf("Parsed: %s\n", buf);
    }

    /* Check for loopback (::1) */
    if (IN6_IS_ADDR_LOOPBACK(&addr)) {
        printf("Address is loopback\n");
    }

    /* Check for link-local (fe80::/10) */
    if (IN6_IS_ADDR_LINKLOCAL(&addr)) {
        printf("Address is link-local\n");
    }

    return 0;
}
```

## Creating an IPv6 TCP Server

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int create_ipv6_server(int port) {
    int sockfd;
    struct sockaddr_in6 server_addr;

    /* Create IPv6 TCP socket */
    sockfd = socket(AF_INET6, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        return -1;
    }

    /* Allow address reuse */
    int opt = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    /* Optionally allow dual-stack (IPv4 + IPv6) by disabling IPV6_V6ONLY */
    int v6only = 0;  /* 0 = accept both IPv4 and IPv6 */
    setsockopt(sockfd, IPPROTO_IPV6, IPV6_V6ONLY, &v6only, sizeof(v6only));

    /* Bind to :: (all IPv6 interfaces) */
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin6_family = AF_INET6;
    server_addr.sin6_port = htons(port);
    server_addr.sin6_addr = in6addr_any;  /* :: */

    if (bind(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind");
        close(sockfd);
        return -1;
    }

    if (listen(sockfd, 5) < 0) {
        perror("listen");
        close(sockfd);
        return -1;
    }

    printf("IPv6 server listening on [::]:5%d\n", port);
    return sockfd;
}

int main(void) {
    int server = create_ipv6_server(8080);
    if (server < 0) return 1;

    while (1) {
        struct sockaddr_in6 client_addr;
        socklen_t addr_len = sizeof(client_addr);

        int client = accept(server, (struct sockaddr *)&client_addr, &addr_len);
        if (client < 0) { perror("accept"); continue; }

        /* Get client IPv6 address string */
        char client_ip[INET6_ADDRSTRLEN];
        inet_ntop(AF_INET6, &client_addr.sin6_addr, client_ip, sizeof(client_ip));

        printf("Connection from [%s]:%d\n", client_ip, ntohs(client_addr.sin6_port));

        const char *msg = "Hello from IPv6 C server\n";
        write(client, msg, strlen(msg));
        close(client);
    }
}
```

## Connecting to an IPv6 Server

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

int connect_ipv6(const char *addr_str, int port) {
    int sockfd;
    struct sockaddr_in6 server_addr;

    sockfd = socket(AF_INET6, SOCK_STREAM, 0);
    if (sockfd < 0) { perror("socket"); return -1; }

    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin6_family = AF_INET6;
    server_addr.sin6_port = htons(port);

    /* Parse the IPv6 address string to binary */
    if (inet_pton(AF_INET6, addr_str, &server_addr.sin6_addr) != 1) {
        fprintf(stderr, "Invalid address: %s\n", addr_str);
        close(sockfd);
        return -1;
    }

    if (connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sockfd);
        return -1;
    }

    return sockfd;
}
```

## Protocol-Independent Code with getaddrinfo

For portable code that handles both IPv4 and IPv6:

```c
#include <netdb.h>

int connect_to_host(const char *host, const char *port) {
    struct addrinfo hints, *res, *p;
    int sockfd = -1;

    memset(&hints, 0, sizeof(hints));
    hints.ai_family   = AF_UNSPEC;    /* Accept IPv4 or IPv6 */
    hints.ai_socktype = SOCK_STREAM;

    if (getaddrinfo(host, port, &hints, &res) != 0) {
        perror("getaddrinfo");
        return -1;
    }

    /* Try each result until one connects */
    for (p = res; p != NULL; p = p->ai_next) {
        sockfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol);
        if (sockfd < 0) continue;

        if (connect(sockfd, p->ai_addr, p->ai_addrlen) == 0) break;
        close(sockfd);
        sockfd = -1;
    }

    freeaddrinfo(res);
    return sockfd;  /* -1 if all connections failed */
}
```

## Conclusion

C IPv6 programming uses `inet_pton(AF_INET6, ...)` to parse addresses, `struct sockaddr_in6` to hold them, and `AF_INET6` socket family for dedicated IPv6 sockets. Use `getaddrinfo()` with `AF_UNSPEC` for portable protocol-independent code that handles both IPv4 and IPv6 without duplication.
