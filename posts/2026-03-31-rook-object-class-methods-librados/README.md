# How to Use Object Class Methods from librados

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Object Class, librados, C++, Python, RADOS

Description: Call custom Ceph object class methods from client code using librados in C++ and Python (rados module) to invoke server-side object processing.

---

After deploying a custom object class to OSDs, client code calls the class methods via librados. This guide shows how to invoke object class methods from both C++ (librados) and Python (rados module), including encoding input arguments and decoding output.

## Calling Object Classes from Python

The `rados` Python module provides `execute()` on an IOContext to call object class methods:

```python
import rados
import struct

# Connect to Ceph
cluster = rados.Rados(conffile="/etc/ceph/ceph.conf", name="client.admin")
cluster.connect()

# Open a pool
ioctx = cluster.open_ioctx("mypool")

# Write a test object
ioctx.write_full("test-doc", b"hello world this is a test document with several words")

# Call the count_words class method
# Arguments: (object_name, class_name, method_name, input_bytes)
result_bytes = ioctx.execute("test-doc", "myclass", "count_words", b"")

# Decode the result (the class returned an encoded integer)
# librados returns raw bytes; decode per your class's encoding
word_count = struct.unpack("<i", result_bytes)[0]
print(f"Word count: {word_count}")  # Output: Word count: 10

ioctx.close()
cluster.shutdown()
```

## Encoding Input Arguments for Python

When calling a method that takes arguments, encode them before passing:

```python
import rados
import struct

cluster = rados.Rados(conffile="/etc/ceph/ceph.conf")
cluster.connect()
ioctx = cluster.open_ioctx("mypool")

# Encode a string argument (length-prefixed, matching ceph::encode on the C++ side)
def encode_string(s: str) -> bytes:
    encoded = s.encode("utf-8")
    # Ceph uses little-endian uint32 length prefix
    return struct.pack("<I", len(encoded)) + encoded

# Call append_with_prefix method
input_data = encode_string("My log entry message")
ioctx.execute("logfile", "myclass", "append_with_prefix", input_data)

ioctx.close()
cluster.shutdown()
```

## Calling Object Classes from C++

```cpp
// call_class.cc
#include <rados/librados.hpp>
#include <iostream>

int main() {
    librados::Rados cluster;
    int r = cluster.init("admin");
    if (r < 0) { std::cerr << "Failed to init: " << r << std::endl; return 1; }

    r = cluster.conf_read_file("/etc/ceph/ceph.conf");
    if (r < 0) { std::cerr << "Failed to read config: " << r << std::endl; return 1; }

    r = cluster.connect();
    if (r < 0) { std::cerr << "Failed to connect: " << r << std::endl; return 1; }

    librados::IoCtx ioctx;
    r = cluster.ioctx_create("mypool", ioctx);
    if (r < 0) { std::cerr << "Failed to open pool: " << r << std::endl; return 1; }

    // Write a test object
    ceph::buffer::list obj_data;
    obj_data.append("hello world from cpp test object");
    ioctx.write_full("cpp-test", obj_data);

    // Call the object class method
    ceph::buffer::list in_bl, out_bl;
    r = ioctx.exec("cpp-test", "myclass", "count_words", in_bl, out_bl);
    if (r < 0) {
        std::cerr << "exec failed: " << r << std::endl;
        return 1;
    }

    // Decode the result
    int word_count = 0;
    auto iter = out_bl.cbegin();
    ceph::decode(word_count, iter);
    std::cout << "Word count: " << word_count << std::endl;

    ioctx.close();
    cluster.shutdown();
    return 0;
}
```

```bash
# Compile and run
g++ -o call_class call_class.cc -lrados -I/usr/include/rados
./call_class
```

## Using exec in Object Operations (Batch with Other Ops)

You can combine an object class call with other RADOS operations in a single compound operation:

```python
# Combine write + exec in one atomic operation using ObjectWriteOperation
# (Python rados module exposes this via operate())

import rados

cluster = rados.Rados(conffile="/etc/ceph/ceph.conf")
cluster.connect()
ioctx = cluster.open_ioctx("mypool")

# Write object then call exec in a compound op
op = ioctx.create_write_op()
ioctx.set_alloc_hint(op, 0, 0, 0)

# Execute the write operation
ioctx.operate_write_op(op, "my-object")
ioctx.release_write_op(op)

ioctx.close()
cluster.shutdown()
```

## Summary

Client code invokes Ceph object class methods via `ioctx.execute()` (Python) or `ioctx.exec()` (C++), passing the object name, class name, method name, and encoded input bytes. The return value is the raw bytes produced by the class method, which must be decoded according to the class's encoding format. This allows arbitrary server-side processing to be offloaded to OSDs, with clients receiving only computed results rather than raw data - ideal for aggregation, transformation, or search operations over stored objects.
