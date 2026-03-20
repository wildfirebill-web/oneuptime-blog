# How to Port IPv4 Socket Applications to IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPv4, Socket Programming, C, Migration, Porting, Networking

Description: Port existing IPv4 socket applications to IPv6 by replacing address structures, updating bind and connect calls, and using getaddrinfo for protocol-independent code.

## Introduction

Porting an IPv4 socket application to IPv6 involves replacing `sockaddr_in` with `sockaddr_in6`, updating address constants, changing socket creation to use `AF_INET6`, and preferably refactoring to use `getaddrinfo()` for protocol independence. This guide walks through the systematic changes needed.

## Summary of Changes

| IPv4 | IPv6 |
|------|------|
| `AF_INET` | `AF_INET6` |
| `struct sockaddr_in` | `struct sockaddr_in6` |
| `struct in_addr` | `struct in6_addr` |
| `sin_family` | `sin6_family` |
| `sin_port` | `sin6_port` |
| `sin_addr` | `sin6_addr` |
| `INADDR_ANY` | `in6addr_any` or `IN6ADDR_ANY_INIT` |
| `INADDR_LOOPBACK` | `in6addr_loopback` |
| `inet_aton()` | `inet_pton(AF_INET6, ...)` |
| `inet_ntoa()` | `inet_ntop(AF_INET6, ...)` |
| `gethostbyname()` | `getaddrinfo()` |
| `INET_ADDRSTRLEN` | `INET6_ADDRSTRLEN` |

## Before: IPv4 Server

```c
/* IPv4 server - BEFORE porting */
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

int create_ipv4_server(int port) {
    /* 1. Create IPv4 socket */
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    int reuse = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    /* 2. Use sockaddr_in */
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(port);
    addr.sin_addr.s_addr = INADDR_ANY;  /* 0.0.0.0 */

    bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(sockfd, 5);

    /* 3. Accept uses sockaddr_in for client */
    struct sockaddr_in client;
    socklen_t len = sizeof(client);
    int client_fd = accept(sockfd, (struct sockaddr *)&client, &len);

    /* 4. Print client IP using inet_ntoa */
    char *ip = inet_ntoa(client.sin_addr);
    printf("Client: %s:%d\n", ip, ntohs(client.sin_port));

    close(client_fd);
    return sockfd;
}
```

## After: IPv6 Server (Direct Port)

```c
/* IPv6 server - AFTER direct porting */
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

int create_ipv6_server(int port) {
    /* 1. Create IPv6 socket (AF_INET → AF_INET6) */
    int sockfd = socket(AF_INET6, SOCK_STREAM, 0);

    int reuse = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    /* Optional: dual-stack to also accept IPv4 */
    int v6only = 0;
    setsockopt(sockfd, IPPROTO_IPV6, IPV6_V6ONLY, &v6only, sizeof(v6only));

    /* 2. Use sockaddr_in6 instead of sockaddr_in */
    struct sockaddr_in6 addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin6_family = AF_INET6;          /* AF_INET → AF_INET6 */
    addr.sin6_port   = htons(port);       /* Same: sin_port → sin6_port */
    addr.sin6_addr   = in6addr_any;       /* INADDR_ANY → in6addr_any */

    bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(sockfd, 5);

    /* 3. Accept uses sockaddr_in6 */
    struct sockaddr_in6 client;
    socklen_t len = sizeof(client);
    int client_fd = accept(sockfd, (struct sockaddr *)&client, &len);

    /* 4. Print client IP using inet_ntop (inet_ntoa → inet_ntop) */
    char ip[INET6_ADDRSTRLEN];   /* INET_ADDRSTRLEN → INET6_ADDRSTRLEN */
    inet_ntop(AF_INET6, &client.sin6_addr, ip, sizeof(ip));
    printf("Client: [%s]:%d\n", ip, ntohs(client.sin6_port));

    close(client_fd);
    return sockfd;
}
```

## Best Practice: Use getaddrinfo (Protocol-Independent)

```c
/* Best approach: rewrite using getaddrinfo for full portability */
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

int create_server(const char *port) {
    struct addrinfo hints, *result;

    memset(&hints, 0, sizeof(hints));
    hints.ai_family   = AF_UNSPEC;     /* Works with IPv4 AND IPv6 */
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags    = AI_PASSIVE;

    if (getaddrinfo(NULL, port, &hints, &result) != 0) return -1;

    int sockfd = socket(result->ai_family, result->ai_socktype, result->ai_protocol);
    int reuse = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    bind(sockfd, result->ai_addr, result->ai_addrlen);
    listen(sockfd, 5);
    freeaddrinfo(result);
    return sockfd;
}
```

## Porting Client Code

```c
/* Before: IPv4 client */
struct sockaddr_in server;
server.sin_family = AF_INET;
server.sin_port = htons(80);
inet_aton("93.184.216.34", &server.sin_addr);
connect(sockfd, (struct sockaddr *)&server, sizeof(server));

/* After: Protocol-independent with getaddrinfo */
struct addrinfo hints, *result;
memset(&hints, 0, sizeof(hints));
hints.ai_family   = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;

getaddrinfo("example.com", "80", &hints, &result);
/* Try each result */
for (struct addrinfo *p = result; p != NULL; p = p->ai_next) {
    sockfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol);
    if (connect(sockfd, p->ai_addr, p->ai_addrlen) == 0) break;
    close(sockfd);
}
freeaddrinfo(result);
```

## Compile and Test

```bash
# Compile with IPv6 support

gcc -o server server.c -Wall -Wextra

# Verify server listens on IPv6
ss -tlnp | grep <port>
# Should show ::: prefix for IPv6

# Test with IPv6 client
nc -6 ::1 <port>
```

## Conclusion

Porting IPv4 socket code to IPv6 requires changing address structures from `sockaddr_in` to `sockaddr_in6`, updating address constants, and replacing `inet_ntoa()`/`inet_aton()` with `inet_ntop()`/`inet_pton()`. The ideal approach is a full rewrite using `getaddrinfo()` with `AF_UNSPEC`, which automatically supports both IPv4 and IPv6 without separate code paths and is more future-proof.
