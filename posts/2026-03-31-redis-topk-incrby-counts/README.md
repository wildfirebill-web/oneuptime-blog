# How to Use TOPK.INCRBY in Redis to Increment Top-K Counts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Top-K, Probabilistic Data Structure

Description: Learn how to use TOPK.INCRBY in Redis to increment item counts in a Top-K structure by arbitrary amounts, enabling weighted frequency tracking in streams.

---

`TOPK.INCRBY` increments the count of one or more items in a Redis Top-K structure by specified amounts. Unlike `TOPK.ADD` which always increments by 1, `TOPK.INCRBY` lets you apply weighted increments - useful when events have different magnitudes, like tracking revenue by product or page views by traffic source weight.

## Basic Syntax

```text
TOPK.INCRBY key item increment [item increment ...]
```

Each `item increment` pair adds the given increment to that item's count in the Top-K structure. The command returns the item that was displaced from the top K (if any), or `nil` if no displacement occurred.

## Basic Example

```bash
# Create a Top-5 structure
127.0.0.1:6379> TOPK.RESERVE revenue:products 5
OK

# Add weighted counts (simulate revenue amounts)
127.0.0.1:6379> TOPK.INCRBY revenue:products "Product A" 150 "Product B" 75 "Product C" 300
1) (nil)
2) (nil)
3) (nil)
```

`nil` results mean no item was bumped out of the top K. If an item is displaced, its key is returned.

## Detecting Rank Changes

```bash
127.0.0.1:6379> TOPK.INCRBY revenue:products "Product D" 500
1) (nil)

# Product D now has 500 - it enters top 5
# If top 5 is full, the lowest-ranked item is displaced and returned
```

## Python Example: Tracking Weighted Revenue

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

r.execute_command("TOPK.RESERVE", "top:revenue", 5)

def record_sale(product_id: str, amount: float) -> None:
    """Record a sale by incrementing the product's revenue count."""
    displaced = r.execute_command(
        "TOPK.INCRBY", "top:revenue", product_id, int(amount)
    )
    if displaced and displaced[0]:
        print(f"Product '{displaced[0]}' dropped out of Top 5")
    else:
        print(f"Recorded ${amount} for {product_id}")

# Simulate sales events
sales = [
    ("widget-pro", 299.99),
    ("gadget-plus", 149.50),
    ("widget-pro", 299.99),
    ("super-tool", 89.00),
    ("widget-pro", 299.99),
    ("gadget-plus", 149.50),
    ("budget-item", 19.99),
]

for product, amount in sales:
    record_sale(product, amount)

top_products = r.execute_command("TOPK.LIST", "top:revenue", "WITHCOUNT")
print("\nTop Revenue Products:")
for i in range(0, len(top_products), 2):
    print(f"  {top_products[i]}: {top_products[i+1]}")
```

## Batch Increment for Analytics Events

```python
import redis
from collections import Counter

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def flush_counters_to_topk(key: str, event_counts: dict) -> None:
    """
    Flush a batch of event counts to Top-K in one command.
    event_counts: dict of {item: count}
    """
    if not event_counts:
        return
    args = []
    for item, count in event_counts.items():
        args.extend([item, count])
    r.execute_command("TOPK.INCRBY", key, *args)

# Simulate collecting events in memory before flushing
event_buffer = [
    "page:/home", "page:/products", "page:/home", "page:/checkout",
    "page:/home", "page:/products", "page:/about", "page:/products"
]

batch_counts = Counter(event_buffer)
r.execute_command("TOPK.RESERVE", "top:pages", 3)
flush_counters_to_topk("top:pages", batch_counts)

result = r.execute_command("TOPK.LIST", "top:pages", "WITHCOUNT")
print("Top pages:", result)
```

## TOPK.INCRBY vs TOPK.ADD

| Feature | TOPK.ADD | TOPK.INCRBY |
|---|---|---|
| Increment amount | Always 1 | Arbitrary positive integer |
| Batch support | Yes (multiple items) | Yes (multiple item/count pairs) |
| Use case | Event counting | Weighted frequency tracking |

## Summary

`TOPK.INCRBY` extends the Top-K structure with weighted increment support, making it ideal for revenue tracking, time-weighted scoring, or any use case where events carry different magnitudes. By batching multiple item/increment pairs in a single call, you can efficiently flush aggregated counts to Redis while keeping your Top-K structure up to date.
