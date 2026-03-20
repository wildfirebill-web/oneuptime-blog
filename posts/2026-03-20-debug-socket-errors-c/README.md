# How to Debug Socket Errors (ECONNREFUSED, ETIMEDOUT, EADDRINUSE) in C

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: C, IPv4, Sockets, Debugging, Errno, POSIX, Networking

Description: Learn how to diagnose and handle common IPv4 socket errors in C - including ECONNREFUSED, ETIMEDOUT, EADDRINUSE, and ECONNRESET - using errno, perror, and strerror.

## Reading Socket Errors

```c
#include <errno.h>
#include <string.h>
#include <stdio.h>

/* After any socket call that returns -1, errno holds the error code */
void check_socket_error(const char *context) {
    /* perror() prints "context: human-readable error message\n" */
    perror(context);

    /* strerror() converts errno to a string without printing */
    fprintf(stderr, "%s failed: [errno %d] %s\n",
            context, errno, strerror(errno));
}
```

## Common Errors and Causes

| errno constant | Value (Linux) | Cause |
|----------------|--------------|-------|
| `ECONNREFUSED` | 111 | No process listening on that port |
| `ETIMEDOUT` | 110 | No response within TCP retry window |
| `EADDRINUSE` | 98 | Port already bound by another socket |
| `ECONNRESET` | 104 | Peer sent RST (abrupt close) |
| `EPIPE` | 32 | Writing to a connection the peer closed |
| `ENETUNREACH` | 101 | No route to host |
| `EINPROGRESS` | 115 | Non-blocking connect in progress (not an error) |
| `EAGAIN` / `EWOULDBLOCK` | 11 / 11 | No data yet (non-blocking) or timeout expired |

## Handling ECONNREFUSED

```c
#include <sys/socket.h>
#include <arpa/inet.h>
#include <errno.h>
#include <stdio.h>
#include <unistd.h>

void connect_and_report(const char *ip, uint16_t port) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(port);
    inet_pton(AF_INET, ip, &addr.sin_addr);

    if (connect(fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        switch (errno) {
        case ECONNREFUSED:
            fprintf(stderr, "%s:%d - connection refused (no server listening)\n",
                    ip, port);
            break;
        case ETIMEDOUT:
            fprintf(stderr, "%s:%d - connection timed out (host unreachable or firewall)\n",
                    ip, port);
            break;
        case ENETUNREACH:
            fprintf(stderr, "%s:%d - network unreachable (no route)\n", ip, port);
            break;
        default:
            perror("connect");
        }
        close(fd);
        return;
    }

    printf("Connected to %s:%d\n", ip, port);
    close(fd);
}
```

## Handling EADDRINUSE

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <errno.h>
#include <stdio.h>

int bind_with_reuse(uint16_t port) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);

    /* SO_REUSEADDR avoids EADDRINUSE after a fast server restart */
    int opt = 1;
    setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {0};
    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port        = htons(port);

    if (bind(fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        if (errno == EADDRINUSE) {
            fprintf(stderr, "Port %d is already in use - "
                            "set SO_REUSEADDR or wait for TIME_WAIT to expire\n", port);
        } else {
            perror("bind");
        }
        return -1;
    }
    return fd;
}
```

## Handling ECONNRESET and EPIPE

```c
#include <signal.h>
#include <errno.h>
#include <stdio.h>

/* Ignore SIGPIPE so send() returns EPIPE instead of killing the process */
void ignore_sigpipe(void) {
    signal(SIGPIPE, SIG_IGN);
}

void safe_send(int fd, const char *buf, size_t len) {
    ssize_t ret = send(fd, buf, len, MSG_NOSIGNAL); /* MSG_NOSIGNAL also suppresses SIGPIPE */
    if (ret < 0) {
        if (errno == EPIPE || errno == ECONNRESET) {
            fprintf(stderr, "Connection broken: peer closed the socket\n");
        } else {
            perror("send");
        }
    }
}

void safe_recv(int fd, char *buf, size_t len) {
    ssize_t ret = recv(fd, buf, len, 0);
    if (ret == 0) {
        printf("Connection closed gracefully by peer\n");
    } else if (ret < 0) {
        if (errno == ECONNRESET) {
            fprintf(stderr, "Connection reset by peer (RST received)\n");
        } else if (errno == EAGAIN || errno == EWOULDBLOCK) {
            fprintf(stderr, "recv timed out - no data\n");
        } else {
            perror("recv");
        }
    }
}
```

## Retrieve Async Connect Error via SO_ERROR

```c
/* After non-blocking connect() → select() returns writable, check actual result */
int get_connect_error(int fd) {
    int       err = 0;
    socklen_t len = sizeof(err);
    getsockopt(fd, SOL_SOCKET, SO_ERROR, &err, &len);
    if (err != 0) {
        fprintf(stderr, "Async connect failed: %s\n", strerror(err));
    }
    return err;  /* 0 = success */
}
```

## Conclusion

Every failed socket call returns `-1` and sets `errno`. Use `perror()` for quick human-readable output and `strerror(errno)` when you need the message as a string in a log. The most frequent errors are: `ECONNREFUSED` (no listener on that port - check the server is running), `EADDRINUSE` (port taken - set `SO_REUSEADDR` or choose a different port), `ETIMEDOUT` (host unreachable or firewall drops - check routing and firewall rules), `ECONNRESET` (peer sent RST - handle gracefully in `recv`/`send` loops), and `EPIPE` (writing to a closed connection - suppress with `MSG_NOSIGNAL` or by ignoring `SIGPIPE`).
