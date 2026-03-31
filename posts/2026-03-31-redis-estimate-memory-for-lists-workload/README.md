# How to Estimate Redis Memory for Lists Workload

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Memory, Capacity Planning, List, DevOps

Description: Learn how to estimate Redis memory for list workloads by understanding quicklist encoding, node size configuration, and per-element storage overhead.

---

Redis lists are used for queues, timelines, and job queues. Their memory consumption is determined by the quicklist encoding, which balances between compact ziplist nodes and efficient linked list traversal.

## How Redis Lists Are Stored

Since Redis 3.2, lists use the `quicklist` encoding - a doubly-linked list of ziplist nodes. Each node holds up to N elements (configurable), compressed or uncompressed.

```bash
redis-cli CONFIG GET list-max-listpack-size
# list-max-listpack-size: 128
```

Each quicklist node holds up to 128 elements by default (negative values mean size limits in bytes).

## Memory Per List Element

```text
Quicklist overhead (base):        ~72 bytes per list key
Per quicklist node:               ~32 bytes node header
Per element in ziplist node:      ~11 bytes overhead + element_size bytes

For a list with 500 elements of 20 bytes each:
  Nodes needed: ceil(500 / 128) = 4 nodes
  Memory = 72 + (4 * 32) + (500 * (11 + 20))
         = 72 + 128 + 15,500
         = ~15,700 bytes (~15.3 KB)
```

## Practical Memory Examples

```bash
# Create a list with 1000 elements
for i in $(seq 1 1000); do redis-cli RPUSH mylist "item_${i}_payload_data"; done

redis-cli LLEN mylist
redis-cli MEMORY USAGE mylist
redis-cli OBJECT ENCODING mylist
# quicklist
```

Typical output: ~35,000-40,000 bytes for 1000 elements of ~20 bytes each.

## Memory Estimation Script

```python
import math

def estimate_list_memory(
    num_lists: int,
    elements_per_list: int,
    avg_element_size_bytes: int,
    node_capacity: int = 128
) -> dict:
    base_overhead = 72  # per list
    node_header = 32    # per quicklist node
    element_overhead = 11  # per element in ziplist

    num_nodes = math.ceil(elements_per_list / node_capacity)
    per_list_bytes = (
        base_overhead
        + num_nodes * node_header
        + elements_per_list * (element_overhead + avg_element_size_bytes)
    )

    total_mb = (per_list_bytes * num_lists) / 1024 / 1024

    return {
        "elements_per_list": elements_per_list,
        "num_nodes_per_list": num_nodes,
        "bytes_per_list": per_list_bytes,
        "total_mb": round(total_mb, 1),
        "total_gb": round(total_mb / 1024, 3),
    }

# 10,000 job queues with 500 items each (30 bytes per item)
result = estimate_list_memory(
    num_lists=10_000,
    elements_per_list=500,
    avg_element_size_bytes=30
)
print(result)
# {'elements_per_list': 500, 'num_nodes_per_list': 4,
#  'bytes_per_list': 20700, 'total_mb': 197.3, 'total_gb': 0.193}
```

## Impact of Node Size Configuration

Larger nodes reduce memory overhead but increase the cost of inserting/removing from the middle:

```bash
# Store up to 256 elements per node (reduces node count, saves memory)
redis-cli CONFIG SET list-max-listpack-size 256
```

```text
With 128 elements/node: 4 nodes for 500 elements
With 256 elements/node: 2 nodes for 500 elements
Memory savings: ~64 bytes per list (modest but multiplies with millions of lists)
```

## Long Lists (>= 512 elements)

For very long lists, consider using a Stream instead, which is more memory-efficient for append-only workloads.

```bash
# Compare memory: list vs stream for 10,000 items
redis-cli MEMORY USAGE mylist_10k
redis-cli MEMORY USAGE mystream_10k
```

## Monitoring List Memory in Production

```bash
# Find the largest lists
redis-cli --scan --pattern "*" | while read key; do
  type=$(redis-cli TYPE "$key")
  if [ "$type" = "list" ]; then
    len=$(redis-cli LLEN "$key")
    mem=$(redis-cli MEMORY USAGE "$key")
    echo "$mem $len $key"
  fi
done | sort -rn | head -10
```

## Summary

Redis list memory is governed by the quicklist structure, with each node holding up to 128 elements by default. Memory per element is approximately 11 bytes overhead plus the element size itself. For large workloads with millions of list keys, use the estimation formula and validate against `MEMORY USAGE` on real data. Consider Streams for append-only high-volume workloads where memory efficiency is critical.
