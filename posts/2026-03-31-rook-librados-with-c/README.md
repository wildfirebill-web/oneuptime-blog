# How to Use librados with C

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, librados, C Programming, RADOS

Description: Learn how to use the librados C API to connect to a Ceph cluster and perform object read, write, and delete operations from a C application.

---

The librados C API provides the most direct access to Ceph's RADOS layer. It is the foundation on which all other bindings are built, offering maximum control with explicit resource management. This guide walks through the essential operations using the C API.

## Installation

Install the development headers:

```bash
# Ubuntu/Debian
sudo apt install librados-dev

# RHEL/CentOS
sudo dnf install librados-devel
```

## Complete Example: Write and Read an Object

```c
#include <rados/librados.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main() {
    rados_t cluster;
    rados_ioctx_t io;
    int ret;

    /* Initialize the cluster handle */
    ret = rados_create(&cluster, "admin");
    if (ret < 0) {
        fprintf(stderr, "Failed to create cluster handle: %d\n", ret);
        return 1;
    }

    /* Read the Ceph configuration */
    ret = rados_conf_read_file(cluster, "/etc/ceph/ceph.conf");
    if (ret < 0) {
        fprintf(stderr, "Failed to read config: %d\n", ret);
        rados_shutdown(cluster);
        return 1;
    }

    /* Connect to the cluster */
    ret = rados_connect(cluster);
    if (ret < 0) {
        fprintf(stderr, "Failed to connect: %d\n", ret);
        rados_shutdown(cluster);
        return 1;
    }

    /* Open an I/O context for the pool */
    ret = rados_ioctx_create(cluster, "mypool", &io);
    if (ret < 0) {
        fprintf(stderr, "Failed to open pool: %d\n", ret);
        rados_shutdown(cluster);
        return 1;
    }

    /* Write an object */
    const char *data = "Hello from librados!";
    ret = rados_write_full(io, "myobj", data, strlen(data));
    if (ret < 0) {
        fprintf(stderr, "Write failed: %d\n", ret);
    } else {
        printf("Write successful\n");
    }

    /* Read the object back */
    char buf[256] = {0};
    ret = rados_read(io, "myobj", buf, sizeof(buf) - 1, 0);
    if (ret < 0) {
        fprintf(stderr, "Read failed: %d\n", ret);
    } else {
        printf("Read %d bytes: %s\n", ret, buf);
    }

    /* Cleanup */
    rados_ioctx_destroy(io);
    rados_shutdown(cluster);
    return 0;
}
```

## Compiling

```bash
gcc -o rados_example rados_example.c -lrados
./rados_example
```

## Key API Functions

| Function                  | Description                         |
|---------------------------|-------------------------------------|
| `rados_create()`          | Create a new cluster handle         |
| `rados_conf_read_file()`  | Load configuration from file        |
| `rados_connect()`         | Connect to the Ceph cluster         |
| `rados_ioctx_create()`    | Open a pool I/O context             |
| `rados_write_full()`      | Write entire object content         |
| `rados_read()`            | Read bytes from an object           |
| `rados_remove()`          | Delete an object                    |
| `rados_ioctx_destroy()`   | Close the I/O context               |
| `rados_shutdown()`        | Disconnect and free the handle      |

## Deleting an Object

```c
ret = rados_remove(io, "myobj");
if (ret == 0) {
    printf("Object deleted\n");
}
```

## Working with Extended Attributes

```c
/* Set an extended attribute */
rados_setxattr(io, "myobj", "content-type", "text/plain", 10);

/* Get an extended attribute */
char xattr_val[64] = {0};
rados_getxattr(io, "myobj", "content-type", xattr_val, sizeof(xattr_val));
printf("xattr: %s\n", xattr_val);
```

## Summary

The librados C API follows a consistent pattern of creating handles, reading configuration, connecting, opening an I/O context, and performing operations. The `rados_write_full`, `rados_read`, and `rados_remove` functions cover the fundamental CRUD operations, while `rados_setxattr` and `rados_getxattr` handle per-object metadata. Always destroy I/O contexts and shut down the cluster handle to avoid resource leaks.
