# How to Create a Multi-Threaded TCP Server in C for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: C, IPv4, TCP, Multi-Threading, POSIX, Networking

Description: Learn how to create a multi-threaded TCP server in C for IPv4 using POSIX threads (pthreads), handling each client connection in a dedicated thread with proper resource cleanup.

## Multi-Threaded Echo Server

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <errno.h>

#define PORT    9000
#define BACKLOG 128
#define BUFSIZE 4096

typedef struct {
    int                client_fd;
    struct sockaddr_in client_addr;
} client_args_t;

void *handle_client(void *arg) {
    client_args_t *ca = (client_args_t *)arg;
    int   fd          = ca->client_fd;
    char  ip[INET_ADDRSTRLEN];
    char  buf[BUFSIZE];
    ssize_t n;

    inet_ntop(AF_INET, &ca->client_addr.sin_addr, ip, sizeof(ip));
    printf("[+] %s:%d (tid=%lu)\n",
           ip, ntohs(ca->client_addr.sin_port), pthread_self());

    free(ca);   /* free heap-allocated arg */

    while ((n = recv(fd, buf, sizeof(buf), 0)) > 0) {
        ssize_t sent = 0;
        while (sent < n) {
            ssize_t s = send(fd, buf + sent, n - sent, 0);
            if (s < 0) goto cleanup;
            sent += s;
        }
    }

cleanup:
    printf("[-] %s closed\n", ip);
    close(fd);
    return NULL;
}

int main(void) {
    int                server_fd;
    struct sockaddr_in server_addr;
    int                opt = 1;

    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) { perror("socket"); return 1; }

    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family      = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port        = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind"); return 1;
    }
    if (listen(server_fd, BACKLOG) < 0) {
        perror("listen"); return 1;
    }

    printf("Multi-threaded TCP server on 0.0.0.0:%d\n", PORT);

    while (1) {
        client_args_t *ca = malloc(sizeof(client_args_t));
        if (!ca) { perror("malloc"); continue; }

        socklen_t len = sizeof(ca->client_addr);
        ca->client_fd = accept(server_fd,
                               (struct sockaddr *)&ca->client_addr, &len);
        if (ca->client_fd < 0) {
            perror("accept");
            free(ca);
            continue;
        }

        pthread_t tid;
        if (pthread_create(&tid, NULL, handle_client, ca) != 0) {
            perror("pthread_create");
            close(ca->client_fd);
            free(ca);
            continue;
        }
        /* Detach: resources freed automatically when thread exits */
        pthread_detach(tid);
    }

    close(server_fd);
    return 0;
}
```

## Compile

```bash
gcc -Wall -Wextra -pthread -o mt_server mt_server.c
./mt_server
```

## Thread Pool Version

```c
#include <pthread.h>
#include <semaphore.h>

#define MAX_THREADS 20

/* Semaphore limits active threads to MAX_THREADS */
sem_t thread_sem;

void *handle_client_pooled(void *arg) {
    handle_client(arg);  /* reuse existing handler */
    sem_post(&thread_sem);  /* release slot */
    return NULL;
}

int main(void) {
    sem_init(&thread_sem, 0, MAX_THREADS);
    /* ... socket setup ... */
    while (1) {
        client_args_t *ca = malloc(sizeof(client_args_t));
        /* ... accept ... */

        sem_wait(&thread_sem);  /* block if at thread limit */
        pthread_t tid;
        pthread_create(&tid, NULL, handle_client_pooled, ca);
        pthread_detach(tid);
    }
}
```

## Conclusion

Multi-threaded TCP servers in C use `pthread_create` to spawn a thread per accepted connection. Pass client data via a heap-allocated struct (not stack!) to avoid race conditions from variable reuse in the accept loop. Call `pthread_detach` to release thread resources automatically when the handler returns. Use a semaphore to cap concurrent threads and prevent resource exhaustion under load. Always `free` the argument struct inside the thread function - not in the main loop, since the thread is running concurrently.
