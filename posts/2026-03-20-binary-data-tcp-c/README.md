# How to Send and Receive Binary Data Over TCP Sockets in C

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: C, IPv4, TCP, Binary, Socket, POSIX, Networking

Description: Learn how to reliably send and receive structured binary data over IPv4 TCP sockets in C using length-prefix framing, network byte order, and loop-based recv/send to handle partial reads.

## Why TCP Requires Framing

TCP is a byte-stream protocol — it does not preserve message boundaries. A single `send()` of 100 bytes may arrive as two `recv()` calls of 60 and 40 bytes. Binary data must be framed so the receiver knows exactly how many bytes constitute one message.

## Helper: Reliable recv and send Loops

```c
#include <sys/socket.h>
#include <unistd.h>
#include <stdint.h>
#include <errno.h>

/* Read exactly 'n' bytes from fd into buf, retrying on partial reads.
   Returns n on success, 0 on EOF, -1 on error. */
ssize_t recvn(int fd, void *buf, size_t n) {
    size_t  received = 0;
    uint8_t *ptr     = (uint8_t *)buf;
    while (received < n) {
        ssize_t r = recv(fd, ptr + received, n - received, 0);
        if (r == 0)  return 0;   /* peer closed */
        if (r < 0)   return -1;  /* error */
        received += (size_t)r;
    }
    return (ssize_t)received;
}

/* Write exactly 'n' bytes from buf to fd, retrying on partial writes.
   Returns n on success, -1 on error. */
ssize_t sendn(int fd, const void *buf, size_t n) {
    size_t        sent = 0;
    const uint8_t *ptr = (const uint8_t *)buf;
    while (sent < n) {
        ssize_t s = send(fd, ptr + sent, n - sent, 0);
        if (s < 0) return -1;
        sent += (size_t)s;
    }
    return (ssize_t)sent;
}
```

## Length-Prefix Framing Protocol

```c
#include <arpa/inet.h>   /* htonl / ntohl */
#include <stdint.h>
#include <string.h>

/* Send a binary message: 4-byte big-endian length followed by payload */
int send_message(int fd, const void *payload, uint32_t payload_len) {
    uint32_t net_len = htonl(payload_len);   /* convert to network byte order */
    if (sendn(fd, &net_len, sizeof(net_len)) < 0) return -1;
    if (sendn(fd, payload,  payload_len)     < 0) return -1;
    return 0;
}

/* Receive a length-prefixed binary message.
   Allocates heap buffer — caller must free().
   Returns payload size on success, -1 on error / EOF. */
ssize_t recv_message(int fd, void **out) {
    uint32_t net_len;
    if (recvn(fd, &net_len, sizeof(net_len)) <= 0) return -1;

    uint32_t payload_len = ntohl(net_len);
    if (payload_len == 0 || payload_len > 4 * 1024 * 1024) return -1;  /* sanity */

    void *buf = malloc(payload_len);
    if (!buf) return -1;

    if (recvn(fd, buf, payload_len) <= 0) { free(buf); return -1; }
    *out = buf;
    return (ssize_t)payload_len;
}
```

## Sending a C Struct as Binary Data

```c
#include <stdint.h>
#include <arpa/inet.h>
#include <string.h>

/* Fixed-width on-wire sensor reading */
typedef struct __attribute__((packed)) {
    uint32_t sensor_id;    /* big-endian on wire */
    int32_t  temperature;  /* millidegrees Celsius, big-endian */
    uint64_t timestamp_ms; /* epoch milliseconds, big-endian */
} sensor_reading_t;

/* Marshal into network byte order before sending */
void marshal_sensor(const sensor_reading_t *src, sensor_reading_t *dst) {
    dst->sensor_id    = htonl(src->sensor_id);
    dst->temperature  = (int32_t)htonl((uint32_t)src->temperature);
    dst->timestamp_ms = htobe64(src->timestamp_ms);  /* Linux-specific */
}

/* Unmarshal after recv */
void unmarshal_sensor(const sensor_reading_t *src, sensor_reading_t *dst) {
    dst->sensor_id    = ntohl(src->sensor_id);
    dst->temperature  = (int32_t)ntohl((uint32_t)src->temperature);
    dst->timestamp_ms = be64toh(src->timestamp_ms);
}
```

## Sender Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int main(void) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(9000);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);
    connect(fd, (struct sockaddr *)&addr, sizeof(addr));

    sensor_reading_t raw = { .sensor_id = 42, .temperature = 23500, .timestamp_ms = 1711000000000ULL };
    sensor_reading_t wire;
    marshal_sensor(&raw, &wire);

    /* Send as a length-prefixed binary frame */
    send_message(fd, &wire, sizeof(wire));
    printf("Sent sensor reading: id=%u temp=%d\n", raw.sensor_id, raw.temperature);

    close(fd);
    return 0;
}
```

## Receiver Example

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {0};
    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port        = htons(9000);
    bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(server_fd, 5);

    int client_fd = accept(server_fd, NULL, NULL);

    void *buf;
    ssize_t len = recv_message(client_fd, &buf);
    if (len == sizeof(sensor_reading_t)) {
        sensor_reading_t wire, local;
        memcpy(&wire, buf, sizeof(wire));
        unmarshal_sensor(&wire, &local);
        printf("Received: id=%u temp=%d ts=%llu\n",
               local.sensor_id, local.temperature,
               (unsigned long long)local.timestamp_ms);
    }
    free(buf);
    close(client_fd);
    close(server_fd);
    return 0;
}
```

## Conclusion

Reliable binary transfer over TCP requires two layers: a framing protocol (length prefix) to delineate message boundaries, and loop-based `send`/`recv` helpers (`sendn`/`recvn`) to handle partial transfers. Always convert multi-byte integer fields to network byte order (`htonl`/`ntohl`, `htobe64`/`be64toh`) before sending and reverse on receipt. Use `__attribute__((packed))` on structs to prevent compiler padding that would corrupt the on-wire layout. Validate the length field before allocating to prevent memory exhaustion from malformed packets.
