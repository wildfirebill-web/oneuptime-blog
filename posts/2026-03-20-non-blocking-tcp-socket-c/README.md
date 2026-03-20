# How to Create a Non-Blocking TCP Socket in C

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: C, IPv4, TCP, Non-Blocking, fcntl, POSIX, Networking

Description: Learn how to create non-blocking IPv4 TCP sockets in C using fcntl() and O_NONBLOCK, and how to handle EAGAIN correctly for recv, send, and accept calls.

## Making a Socket Non-Blocking

```c
#include <fcntl.h>
#include <unistd.h>
#include <sys/socket.h>
#include <errno.h>
#include <stdio.h>

/* Set a file descriptor to non-blocking mode.
   Returns 0 on success, -1 on error. */
int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags < 0) { perror("fcntl F_GETFL"); return -1; }
    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) < 0) {
        perror("fcntl F_SETFL"); return -1;
    }
    return 0;
}

/* Restore blocking mode */
int set_blocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags < 0) return -1;
    return fcntl(fd, F_SETFL, flags & ~O_NONBLOCK);
}
```

## Creating a Non-Blocking Socket Directly

```c
#include <sys/socket.h>

/* SOCK_NONBLOCK flag (Linux 2.6.27+) sets O_NONBLOCK atomically at creation */
int fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
```

## Non-Blocking accept()

```c
#include <sys/socket.h>
#include <errno.h>

/* In a non-blocking server, accept() returns EAGAIN when no connections are pending.
   Accept all pending connections in a loop before returning to the event loop. */
void accept_all(int server_fd) {
    while (1) {
        struct sockaddr_in caddr;
        socklen_t          clen = sizeof(caddr);

        /* accept4() with SOCK_NONBLOCK sets the new socket non-blocking atomically */
        int cfd = accept4(server_fd, (struct sockaddr *)&caddr, &clen, SOCK_NONBLOCK);
        if (cfd < 0) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                break;   /* no more pending connections */
            }
            perror("accept4");
            break;
        }
        /* handle cfd (add to epoll/select, etc.) */
    }
}
```

## Non-Blocking recv()

```c
#include <errno.h>

/* In non-blocking mode, recv() returns -1 with EAGAIN if no data is available.
   With edge-triggered epoll, drain the socket completely before returning. */
void read_all(int fd, char *buf, size_t bufsz) {
    while (1) {
        ssize_t n = recv(fd, buf, bufsz, 0);
        if (n > 0) {
            /* process n bytes */
        } else if (n == 0) {
            /* peer closed */
            break;
        } else {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                break;   /* no more data right now */
            }
            perror("recv");
            break;
        }
    }
}
```

## Non-Blocking send()

```c
#include <errno.h>

/* Non-blocking send() may write fewer bytes than requested (EAGAIN when buffer full).
   Retry with the unsent remainder. */
ssize_t send_nonblocking(int fd, const char *buf, size_t len) {
    size_t sent = 0;
    while (sent < len) {
        ssize_t s = send(fd, buf + sent, len - sent, 0);
        if (s > 0) {
            sent += (size_t)s;
        } else if (s < 0) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                /* Socket send buffer full — register EPOLLOUT and retry later */
                return (ssize_t)sent;
            }
            perror("send");
            return -1;
        }
    }
    return (ssize_t)sent;
}
```

## Non-Blocking connect()

```c
#include <sys/select.h>
#include <sys/time.h>
#include <arpa/inet.h>

/* Non-blocking connect returns EINPROGRESS immediately.
   Use select()/epoll() to wait for the socket to become writable. */
int connect_nonblocking(const char *ip, uint16_t port) {
    int fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(port);
    inet_pton(AF_INET, ip, &addr.sin_addr);

    if (connect(fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        if (errno != EINPROGRESS) {
            perror("connect"); return -1;
        }
    }

    /* Wait up to 5 seconds for the connection to complete */
    fd_set wfds;
    FD_ZERO(&wfds);
    FD_SET(fd, &wfds);
    struct timeval tv = { .tv_sec = 5, .tv_usec = 0 };
    int rc = select(fd + 1, NULL, &wfds, NULL, &tv);

    if (rc <= 0) { close(fd); return -1; }

    /* Verify connect success via SO_ERROR */
    int err = 0;
    socklen_t elen = sizeof(err);
    getsockopt(fd, SOL_SOCKET, SO_ERROR, &err, &elen);
    if (err) { errno = err; perror("connect async"); close(fd); return -1; }

    return fd;   /* connected, still non-blocking */
}
```

## Blocking vs Non-Blocking Comparison

| Behavior | Blocking | Non-Blocking |
|----------|---------|--------------|
| `recv()` with no data | Waits indefinitely | Returns -1, errno=EAGAIN |
| `send()` with full buffer | Waits for space | Returns -1, errno=EAGAIN |
| `accept()` with no clients | Waits | Returns -1, errno=EAGAIN |
| `connect()` in progress | Waits for handshake | Returns -1, errno=EINPROGRESS |

## Conclusion

Switch a socket to non-blocking mode with `fcntl(fd, F_SETFL, flags | O_NONBLOCK)` or use `SOCK_NONBLOCK` at creation time (Linux). In non-blocking mode, `recv()`, `send()`, and `accept()` return `-1` with `errno == EAGAIN`/`EWOULDBLOCK` instead of blocking — this is not an error, just a signal to retry later. Pair non-blocking sockets with an event-notification mechanism (`select`, `poll`, or `epoll`) to know when a socket is ready before attempting I/O. For `connect()`, detect `EINPROGRESS`, wait for writability via `select`/`epoll`, then confirm success by reading `SO_ERROR`.
