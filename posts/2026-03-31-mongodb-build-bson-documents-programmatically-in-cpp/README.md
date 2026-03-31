# How to Build BSON Documents Programmatically in C++

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, C++, BSON, Document, Builder

Description: Learn how to construct BSON documents in C++ using the bsoncxx stream and basic builders, including nested documents and arrays.

---

## Introduction

BSON (Binary JSON) is MongoDB's wire format. The `bsoncxx` library bundled with the mongocxx driver provides two builder APIs: a stream builder using `<<` operators for composing documents inline, and a basic builder using explicit append calls for programmatic construction. Understanding both is essential for building dynamic queries and documents.

## Includes

```cpp
#include <bsoncxx/builder/stream/document.hpp>
#include <bsoncxx/builder/stream/array.hpp>
#include <bsoncxx/builder/basic/document.hpp>
#include <bsoncxx/builder/basic/array.hpp>
#include <bsoncxx/builder/basic/kvp.hpp>
#include <bsoncxx/json.hpp>
#include <bsoncxx/types.hpp>

using bsoncxx::builder::stream::document;
using bsoncxx::builder::stream::array;
using bsoncxx::builder::stream::open_document;
using bsoncxx::builder::stream::close_document;
using bsoncxx::builder::stream::open_array;
using bsoncxx::builder::stream::close_array;
using bsoncxx::builder::stream::finalize;
```

## Stream Builder - Flat Document

```cpp
auto doc = document{}
    << "name"     << "Laptop"
    << "price"    << 999.99
    << "inStock"  << true
    << "quantity" << 42
    << finalize;

std::cout << bsoncxx::to_json(doc.view()) << std::endl;
// {"name": "Laptop", "price": 999.99, "inStock": true, "quantity": 42}
```

## Stream Builder - Nested Document

```cpp
auto nested = document{}
    << "title" << "Product"
    << "specs" << open_document
        << "ram"    << "16GB"
        << "storage" << "512GB SSD"
    << close_document
    << finalize;
```

## Stream Builder - Arrays

```cpp
auto with_array = document{}
    << "name" << "Laptop"
    << "tags" << open_array
        << "portable" << "work" << "wireless"
    << close_array
    << finalize;
```

## Basic Builder - Programmatic Construction

Use the basic builder when field names or values are determined at runtime:

```cpp
#include <bsoncxx/builder/basic/document.hpp>
using bsoncxx::builder::basic::kvp;
using bsoncxx::builder::basic::make_document;
using bsoncxx::builder::basic::make_array;

// Dynamic document from a map
std::map<std::string, std::string> metadata = {
    { "author", "Alice" }, { "lang", "en" }
};

bsoncxx::builder::basic::document builder{};
for (auto& [key, val] : metadata) {
    builder.append(kvp(key, val));
}
auto doc = builder.extract();
```

## ObjectId and Dates

```cpp
#include <bsoncxx/oid.hpp>
#include <bsoncxx/types.hpp>
#include <chrono>

bsoncxx::oid oid{};
auto now = bsoncxx::types::b_date{
    std::chrono::system_clock::now()
};

auto doc = document{}
    << "_id"        << oid
    << "created_at" << now
    << finalize;
```

## Converting JSON String to BSON

```cpp
auto doc = bsoncxx::from_json(R"({"name": "Mouse", "price": 29.99})");
std::cout << bsoncxx::to_json(doc.view()) << std::endl;
```

## Summary

The `bsoncxx` stream builder uses `<<` operators and sentinel types (`open_document`, `close_document`, `finalize`) to compose BSON inline. The basic builder's `append(kvp(...))` pattern suits runtime-dynamic construction. Use `bsoncxx::from_json` to parse JSON strings into BSON and `bsoncxx::to_json` to debug document contents. Both builders produce `bsoncxx::document::value` which can be passed directly to mongocxx operations.
