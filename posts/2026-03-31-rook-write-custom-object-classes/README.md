# How to Write Custom Object Classes for Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Object Class, RADOS, C++, Extension, Plugin

Description: Write custom Ceph object class plugins in C++ to implement server-side data processing logic that runs directly on OSDs, reducing client-OSD round-trips.

---

Ceph object classes are C++ plugins that run inside OSD processes, allowing you to push computation to the storage layer. Instead of reading data to the client and processing it, an object class method executes on the OSD where the data lives, returning only the result.

## Why Write Object Classes

- **Reduce network traffic**: Process data where it lives, send only results
- **Atomic operations**: Read-modify-write in a single OSD operation
- **Custom data structures**: Implement counters, inverted indexes, or domain-specific formats
- **Performance**: Avoid serialization overhead for data that stays on the OSD

## Object Class Structure

An object class is a shared library (`.so`) placed in the OSD class directory. It exports a `__cls_init()` function that registers methods:

```cpp
// myclass.cc
#include "objclass/objclass.h"
#include <string>

CLS_VER(1, 0)
CLS_NAME(myclass)

// Method handler: reads the object and returns a word count
static int count_words(cls_method_context_t hctx,
                        ceph::buffer::list *in,
                        ceph::buffer::list *out) {
    ceph::buffer::list data;
    int r = cls_cxx_read(hctx, 0, 0, &data);
    if (r < 0) return r;

    std::string content(data.c_str(), data.length());
    int words = 0;
    bool in_word = false;
    for (char c : content) {
        if (std::isspace(c)) {
            in_word = false;
        } else if (!in_word) {
            in_word = true;
            ++words;
        }
    }

    // Encode the result into the output buffer
    ceph::encode(words, *out);
    return 0;
}

// Append method: append data atomically with a length prefix
static int append_with_prefix(cls_method_context_t hctx,
                               ceph::buffer::list *in,
                               ceph::buffer::list *out) {
    std::string to_append;
    auto iter = in->cbegin();
    ceph::decode(to_append, iter);

    // Prepend current timestamp
    std::string entry = "[" + std::to_string(time(nullptr)) + "] " + to_append + "\n";
    ceph::buffer::list entry_bl;
    entry_bl.append(entry);

    // Append to the object atomically
    int r = cls_cxx_write_full(hctx, &entry_bl);
    return r;
}

void __cls_init() {
    CLS_LOG(0, "loading myclass");

    cls_handle_t h_class;
    cls_method_handle_t h_count_words;
    cls_method_handle_t h_append_with_prefix;

    cls_register("myclass", &h_class);

    cls_register_cxx_method(h_class, "count_words",
                             CLS_METHOD_RD,
                             count_words, &h_count_words);

    cls_register_cxx_method(h_class, "append_with_prefix",
                             CLS_METHOD_RD | CLS_METHOD_WR,
                             append_with_prefix, &h_append_with_prefix);
}
```

## Available OSD Primitives

Within an object class method, you have access to:

```cpp
// Read the object data
cls_cxx_read(hctx, offset, length, &bl);

// Write/overwrite the entire object
cls_cxx_write_full(hctx, &bl);

// Write at a specific offset
cls_cxx_write(hctx, offset, length, &bl);

// Get/set extended attributes
cls_cxx_getxattr(hctx, "myattr", &bl);
cls_cxx_setxattr(hctx, "myattr", &bl);

// Get object size and metadata
cls_cxx_stat(hctx, &size, &mtime);

// Logging
CLS_LOG(10, "debug message: %s", value.c_str());
CLS_ERR("error: %d", errcode);
```

## CMakeLists.txt for the Object Class

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.10)
project(myclass)

find_package(Ceph REQUIRED)

add_library(cls_myclass SHARED myclass.cc)
target_link_libraries(cls_myclass ${CEPH_LIBRARIES})
set_target_properties(cls_myclass PROPERTIES PREFIX "")
install(TARGETS cls_myclass DESTINATION /usr/lib/rados-classes/)
```

## Summary

Ceph object classes are C++ shared libraries that extend OSD functionality with custom read/write/modify operations. They register methods via `__cls_init()` and use OSD primitives like `cls_cxx_read` and `cls_cxx_write_full` to manipulate object data atomically. Well-designed object classes reduce client-OSD round-trips by keeping computation close to data, making them ideal for aggregate operations, atomic counters, and custom data structure updates that would otherwise require read-modify-write cycles over the network.
