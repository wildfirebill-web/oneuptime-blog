# How to Use VINFO in Redis for Vector Set Information

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Vector Sets, Metadata, Diagnostics, Index Management

Description: Learn how to use VINFO in Redis to inspect Vector Set metadata including size, dimensions, quantization type, and HNSW graph parameters.

---

## What Is VINFO?

`VINFO` returns detailed metadata about a Redis Vector Set. It provides information about the number of elements, vector dimensions, quantization type, and the internal HNSW (Hierarchical Navigable Small World) graph parameters used for approximate nearest-neighbor search.

This command is essential for capacity planning, debugging, and understanding the state of your vector indexes.

## Syntax

```text
VINFO key
```

## Sample Output

```bash
VADD my_vectors item1 VALUES 4 0.1 0.2 0.3 0.4
VADD my_vectors item2 VALUES 4 0.5 0.6 0.7 0.8
VADD my_vectors item3 VALUES 4 0.9 0.8 0.7 0.6

VINFO my_vectors
# 1) "attributes-count"
# 2) (integer) 0
# 3) "current-elements"
# 4) (integer) 3
# 5) "deleted-elements"
# 6) (integer) 0
# 7) "max-elements"
# 8) (integer) 0
# 9) "quant-type"
# 10) "int8"
# 11) "projection-input-dim"
# 12) (integer) 0
# 13) "vector-dim"
# 14) (integer) 4
# 15) "max-level"
# 16) (integer) 1
# 17) "entry-point"
# 18) "item1"
# 19) "memory-usage"
# 20) (integer) 12345
```

## Understanding Each Field

| Field | Description |
|-------|-------------|
| current-elements | Number of active vectors |
| deleted-elements | Vectors marked for deletion (tombstones) |
| vector-dim | Dimensionality of each vector |
| quant-type | Quantization method: int8, bf16, fp32, or none |
| max-level | HNSW graph height |
| entry-point | ID of the entry-point node in the HNSW graph |
| memory-usage | Approximate memory in bytes |
| attributes-count | Number of elements with custom attributes |

## Python Example: Vector Index Inspector

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def parse_vinfo(key: str) -> dict:
    """Parse VINFO output into a dictionary."""
    try:
        raw = r.execute_command("VINFO", key)
        return dict(zip(raw[::2], raw[1::2]))
    except Exception:
        return {}

def print_index_report(key: str):
    """Print a human-readable report for a Vector Set."""
    info = parse_vinfo(key)
    if not info:
        print(f"Key '{key}' not found or is not a Vector Set")
        return

    current = int(info.get("current-elements", 0))
    deleted = int(info.get("deleted-elements", 0))
    dim = int(info.get("vector-dim", 0))
    quant = info.get("quant-type", "unknown")
    mem_bytes = int(info.get("memory-usage", 0))
    mem_kb = mem_bytes / 1024

    print(f"Vector Index: {key}")
    print(f"  Elements:      {current:,} active, {deleted} deleted")
    print(f"  Dimensions:    {dim}")
    print(f"  Quantization:  {quant}")
    print(f"  HNSW level:    {info.get('max-level', 'N/A')}")
    print(f"  Memory usage:  {mem_kb:.1f} KB ({mem_bytes:,} bytes)")
    if current > 0 and mem_bytes > 0:
        bytes_per_vec = mem_bytes / current
        print(f"  Per vector:    {bytes_per_vec:.0f} bytes")

# Setup and test
r.execute_command("VADD", "products:vectors", "FP32",
                  "prod:1001", "VALUES", "8",
                  "0.1", "0.2", "0.3", "0.4", "0.5", "0.6", "0.7", "0.8")
r.execute_command("VADD", "products:vectors", "FP32",
                  "prod:1002", "VALUES", "8",
                  "0.9", "0.8", "0.7", "0.6", "0.5", "0.4", "0.3", "0.2")

print_index_report("products:vectors")
```

## Capacity Planning with VINFO

Use memory-usage to estimate storage requirements:

```python
def estimate_full_capacity(key: str, target_elements: int) -> dict:
    """Estimate memory needed to hold target_elements vectors."""
    info = parse_vinfo(key)
    current = int(info.get("current-elements", 0))
    mem_bytes = int(info.get("memory-usage", 0))

    if current == 0:
        return {"error": "no elements yet, cannot estimate"}

    bytes_per_element = mem_bytes / current
    estimated_total = bytes_per_element * target_elements

    return {
        "current_elements": current,
        "current_memory_mb": mem_bytes / (1024 * 1024),
        "target_elements": target_elements,
        "estimated_memory_mb": estimated_total / (1024 * 1024),
        "bytes_per_element": bytes_per_element
    }

estimate = estimate_full_capacity("products:vectors", 1_000_000)
print(f"Estimated memory for 1M vectors: {estimate['estimated_memory_mb']:.0f} MB")
```

## Checking Index Health

```python
def is_index_healthy(key: str) -> dict:
    """Check for signs of index health issues."""
    info = parse_vinfo(key)
    current = int(info.get("current-elements", 0))
    deleted = int(info.get("deleted-elements", 0))

    health = {"status": "ok", "warnings": []}

    if current == 0:
        health["status"] = "empty"
        health["warnings"].append("Index has no elements")

    if deleted > 0:
        deletion_ratio = deleted / (current + deleted)
        if deletion_ratio > 0.2:
            health["status"] = "degraded"
            health["warnings"].append(
                f"High deletion ratio: {deletion_ratio:.1%} ({deleted} deleted)"
            )

    return health

health = is_index_healthy("products:vectors")
print(f"Health: {health['status']}")
for w in health["warnings"]:
    print(f"  Warning: {w}")
```

## Comparing VINFO, VCARD, and VDIM

| Command | Returns | Use Case |
|---------|---------|----------|
| VINFO | Full metadata | Detailed diagnostics, capacity planning |
| VCARD | Element count only | Quick size check |
| VDIM | Dimension only | Compatibility validation |

## Summary

`VINFO` provides comprehensive metadata about a Redis Vector Set including element count, dimensions, quantization type, HNSW graph structure, and memory usage. Use it for capacity planning, health monitoring, and debugging vector index behavior. The memory-usage field enables accurate storage estimation for scaling decisions, while deleted-elements reveals fragmentation that may affect search performance.
