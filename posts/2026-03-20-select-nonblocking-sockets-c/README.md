# How to Use the select() Function for Non-Blocking IPv4 Sockets in C

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: C, IPv4, SELECT, Non-Blocking, POSIX, Networking

Description: Learn how to use the POSIX select() function to multiplex multiple IPv4 TCP connections in a single-threaded C server, avoiding blocking on any one socket.

## How select() Works

`select()` monitors a set of file descriptors for readability, writability, and exceptional conditions. It blocks until at least one fd is ready, then returns which fds can be operated on without blocking.

```c
int select(int nfds,
           fd_set *readfds,    /* watch for readable */
           fd_set *writefds,   /* watch for writable */
           fd_set *exceptfds,  /* watch for exceptions */
           struct timeval *timeout);
```

## Echo Server with select()

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/select.h>
#include <sys/socket.h>

#define PORT     9000
#define MAX_FDS  FD_SETSIZE   /* typically 1024 */
#define BUFSIZE  4096

int main(void) {
    int    server_fd, client_fds[MAX_FDS];
    int    max_fd, i;
    fd_set read_set;
    char   buf[BUFSIZE];

    /* Initialize client array */
    for (i = 0; i < MAX_FDS; i++) client_fds[i] = -1;

    /* Create and bind server socket */
    struct sockaddr_in addr;
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    memset(&addr, 0, sizeof(addr));
    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port        = htons(PORT);
    bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(server_fd, 10);

    printf("select() echo server on 0.0.0.0:%d\n", PORT);
    max_fd = server_fd;

    while (1) {
        /* Re-build fd_set every iteration (select modifies it) */
        FD_ZERO(&read_set);
        FD_SET(server_fd, &read_set);

        for (i = 0; i < MAX_FDS; i++) {
            if (client_fds[i] != -1) {
                FD_SET(client_fds[i], &read_set);
                if (client_fds[i] > max_fd)
                    max_fd = client_fds[i];
            }
        }

        /* Block until any fd is readable */
        int ready = select(max_fd + 1, &read_set, NULL, NULL, NULL);
        if (ready < 0) { perror("select"); break; }

        /* New connection */
        if (FD_ISSET(server_fd, &read_set)) {
            struct sockaddr_in caddr;
            socklen_t len = sizeof(caddr);
            int cfd = accept(server_fd, (struct sockaddr *)&caddr, &len);
            if (cfd >= 0) {
                char ip[INET_ADDRSTRLEN];
                inet_ntop(AF_INET, &caddr.sin_addr, ip, sizeof(ip));
                printf("[+] %s:%d (fd=%d)\n", ip, ntohs(caddr.sin_port), cfd);
                /* Find empty slot */
                for (i = 0; i < MAX_FDS; i++) {
                    if (client_fds[i] == -1) { client_fds[i] = cfd; break; }
                }
            }
        }

        /* Data from clients */
        for (i = 0; i < MAX_FDS; i++) {
            int cfd = client_fds[i];
            if (cfd == -1 || !FD_ISSET(cfd, &read_set)) continue;

            ssize_t n = recv(cfd, buf, sizeof(buf), 0);
            if (n <= 0) {
                /* Client closed or error */
                printf("[-] fd=%d closed\n", cfd);
                close(cfd);
                client_fds[i] = -1;
            } else {
                send(cfd, buf, n, 0);   /* echo */
            }
        }
    }

    close(server_fd);
    return 0;
}
```

## Compile and Test

```bash
gcc -Wall -o select_server select_server.c
./select_server

# Test with multiple clients

for i in 1 2 3; do
    echo "client $i" | nc -q1 127.0.0.1 9000 &
done
```

## select() Limitations

| Limitation | Details |
|-----------|---------|
| fd limit | `FD_SETSIZE` = 1024 by default |
| O(n) scan | Must scan all fds every call |
| fd_set is modified | Must rebuild before each call |
| Linux alternative | `poll()` (no fd limit) or `epoll()` (O(1)) |

## Conclusion

`select()` is the portable way to multiplex multiple sockets in a single thread. Rebuild the `fd_set` before every call since `select()` modifies it in place. Track client fds in an array and scan for ready fds after `select()` returns. The hard limit of `FD_SETSIZE` (1024) makes `select()` unsuitable for high-connection-count servers - use `poll()` for unlimited fds or `epoll()` (Linux) for O(1) scalability. `select()` is still valuable for simple tools, test code, and portable applications.
