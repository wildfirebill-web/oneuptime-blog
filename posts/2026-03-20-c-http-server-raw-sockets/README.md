# How to Implement a Simple HTTP Server Using Raw Sockets in C

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: C, Sockets, HTTP, IPv4, Networking, Systems Programming

Description: Build a minimal HTTP/1.0 server in C using raw POSIX sockets that accepts TCP connections on IPv4, parses request lines, and returns HTTP responses.

## Introduction

Understanding how HTTP works at the socket level gives you insight into every web framework and server. Building a minimal HTTP server in C using raw sockets strips away the abstraction and shows exactly what happens when a browser connects to port 80.

## Complete HTTP Server

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8080
#define BUFFER_SIZE 4096

void handle_client(int client_fd) {
    char buffer[BUFFER_SIZE];
    ssize_t bytes = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
    if (bytes <= 0) { close(client_fd); return; }
    buffer[bytes] = '\0';

    char method[8], path[256], proto[16];
    sscanf(buffer, "%7s %255s %15s", method, path, proto);
    printf("Request: %s %s\n", method, path);

    const char *body = "<html><body><h1>Hello from C!</h1></body></html>";
    char response[512];
    snprintf(response, sizeof(response),
        "HTTP/1.0 200 OK\r\nContent-Type: text/html\r\n"
        "Content-Length: %zu\r\nConnection: close\r\n\r\n%s",
        strlen(body), body);

    send(client_fd, response, strlen(response), 0);
    close(client_fd);
}

int main(void) {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_addr.s_addr = INADDR_ANY,
        .sin_port = htons(PORT)
    };
    bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(server_fd, 10);
    printf("HTTP server listening on port %d\n", PORT);

    while (1) {
        struct sockaddr_in client_addr;
        socklen_t len = sizeof(client_addr);
        int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &len);
        if (client_fd < 0) continue;
        handle_client(client_fd);
    }
    return 0;
}
```

## Compile and Test

```bash
gcc -o http_server http_server.c
./http_server &
curl http://127.0.0.1:8080/
```

## Serving Static Files

Extend `handle_client` to open local files based on the parsed path and stream their contents with a correct `Content-Length` header. Return a 404 response when the file does not exist.

## Conclusion

A raw-socket HTTP server in C demonstrates TCP socket lifecycle, HTTP framing, and response construction without any library magic. While production servers use non-blocking I/O and thread pools, this sequential model is an excellent learning foundation.
