# How to Handle Multiple Connections with epoll on Linux in C

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: C, IPv4, epoll, Linux, Non-Blocking, Networking

Description: Learn how to use the Linux epoll API to handle thousands of concurrent IPv4 TCP connections in a single-threaded C server with O(1) event notification.

## epoll API Overview

| Function | Purpose |
|----------|---------|
| `epoll_create1(0)` | Create epoll instance |
| `epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev)` | Add fd to watch list |
| `epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev)` | Modify interest flags |
| `epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL)` | Remove fd |
| `epoll_wait(epfd, events, maxevents, timeout)` | Wait for events |

## epoll Echo Server

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <sys/socket.h>

#define PORT       9000
#define MAX_EVENTS 1024
#define BUFSIZE    4096

static int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main(void) {
    int    server_fd, epfd;
    struct epoll_event event, events[MAX_EVENTS];
    struct sockaddr_in server_addr;
    int    opt = 1;

    /* Create non-blocking server socket */
    server_fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family      = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port        = htons(PORT);
    bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
    listen(server_fd, SOMAXCONN);

    /* Create epoll instance */
    epfd = epoll_create1(0);

    /* Register server socket for read events */
    event.events  = EPOLLIN;
    event.data.fd = server_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &event);

    printf("epoll server on 0.0.0.0:%d\n", PORT);

    char buf[BUFSIZE];

    while (1) {
        int n = epoll_wait(epfd, events, MAX_EVENTS, -1);

        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;

            if (fd == server_fd) {
                /* Accept all pending connections */
                while (1) {
                    struct sockaddr_in caddr;
                    socklen_t          clen = sizeof(caddr);
                    int cfd = accept4(server_fd,
                                      (struct sockaddr *)&caddr, &clen,
                                      SOCK_NONBLOCK);
                    if (cfd < 0) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) break;
                        perror("accept4");
                        break;
                    }
                    char ip[INET_ADDRSTRLEN];
                    inet_ntop(AF_INET, &caddr.sin_addr, ip, sizeof(ip));
                    printf("[+] %s:%d (fd=%d)\n", ip, ntohs(caddr.sin_port), cfd);

                    event.events  = EPOLLIN | EPOLLET;  /* edge-triggered */
                    event.data.fd = cfd;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &event);
                }
            } else if (events[i].events & (EPOLLERR | EPOLLHUP)) {
                printf("[-] fd=%d hung up\n", fd);
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
            } else if (events[i].events & EPOLLIN) {
                /* Edge-triggered: must drain all available data */
                while (1) {
                    ssize_t len = recv(fd, buf, sizeof(buf), 0);
                    if (len > 0) {
                        send(fd, buf, len, 0);  /* echo */
                    } else if (len == 0) {
                        /* Client closed connection */
                        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                        close(fd);
                        break;
                    } else {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) break;
                        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                        close(fd);
                        break;
                    }
                }
            }
        }
    }

    close(epfd);
    close(server_fd);
    return 0;
}
```

## Compile

```bash
gcc -Wall -Wextra -o epoll_server epoll_server.c
./epoll_server
```

## Edge vs Level Triggered

```c
/* Level-triggered (default): fires as long as data is available */
event.events = EPOLLIN;

/* Edge-triggered: fires only when new data arrives */
event.events = EPOLLIN | EPOLLET;
/* With ET, you MUST read until EAGAIN before returning to epoll_wait */
```

## Conclusion

`epoll` on Linux provides O(1) event notification regardless of the number of monitored file descriptors, making it suitable for servers with tens of thousands of connections. Use `SOCK_NONBLOCK` and `EPOLLET` (edge-triggered) for maximum performance. In edge-triggered mode, drain the socket completely in a `recv` loop until `EAGAIN` — otherwise the event won't fire again for the remaining data. Use `accept4()` with `SOCK_NONBLOCK` to create non-blocking accepted sockets atomically.
