# How to Use ClickHouse C++ Client Library

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, C++, Client Library, Native Protocol, Data Ingestion

Description: Learn how to use the official ClickHouse C++ client library to connect, query, and insert data using the native binary protocol.

---

The ClickHouse C++ client library (`clickhouse-cpp`) communicates over the native binary protocol on port 9000. It is the lowest-level official client and is suitable for high-performance applications written in C++ that need minimal overhead.

## Installation

Clone and build from source:

```bash
git clone https://github.com/ClickHouse/clickhouse-cpp.git
cd clickhouse-cpp
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```

Or use vcpkg:

```bash
vcpkg install clickhouse-cpp
```

## Connecting to ClickHouse

```cpp
#include <clickhouse/client.h>
using namespace clickhouse;

int main() {
    Client client(ClientOptions()
        .SetHost("localhost")
        .SetPort(9000)
        .SetUser("default")
        .SetPassword("")
        .SetDefaultDatabase("default")
    );
    return 0;
}
```

## Running a SELECT Query

```cpp
client.Select("SELECT number, number * number AS sq FROM numbers(5)",
    [](const Block& block) {
        for (size_t i = 0; i < block.GetRowCount(); ++i) {
            auto num = block[0]->As<ColumnUInt64>()->At(i);
            auto sq  = block[1]->As<ColumnUInt64>()->At(i);
            std::cout << num << " -> " << sq << "\n";
        }
    }
);
```

The callback is invoked once per block. Blocks arrive in streaming fashion so you can process large result sets with constant memory.

## Inserting Data

Build a `Block` with typed columns and call `Insert`:

```cpp
auto col_id   = std::make_shared<ColumnUInt64>();
auto col_name = std::make_shared<ColumnString>();

col_id->Append(1); col_name->Append("alice");
col_id->Append(2); col_name->Append("bob");

Block block;
block.AppendColumn("id",   col_id);
block.AppendColumn("name", col_name);

client.Insert("users", block);
```

## Executing DDL

Use `Execute` for statements that return no result set:

```cpp
client.Execute(R"(
    CREATE TABLE IF NOT EXISTS users (
        id   UInt64,
        name String
    ) ENGINE = MergeTree()
    ORDER BY id
)");
```

## Error Handling

```cpp
try {
    client.Select("SELECT * FROM nonexistent_table", [](const Block&) {});
} catch (const std::exception& e) {
    std::cerr << "ClickHouse error: " << e.what() << "\n";
}
```

## CMake Integration

```text
find_package(clickhouse-cpp REQUIRED)
target_link_libraries(my_app PRIVATE clickhouse-cpp-lib)
```

## Summary

The ClickHouse C++ client library provides low-level, high-performance access to ClickHouse via the native protocol. Use typed column objects to build blocks for insertion, and lambda callbacks to process streaming SELECT results. It is best suited for latency-sensitive C++ applications or custom data pipeline components.
