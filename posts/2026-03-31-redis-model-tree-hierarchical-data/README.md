# How to Model Tree/Hierarchical Data in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Data Modeling, Tree, Hierarchy

Description: Model tree and hierarchical data structures in Redis using parent references, child sets, and path encoding for categories, org charts, and file systems.

---

Redis does not have a native tree type, but you can model hierarchical data effectively using combinations of hashes, sets, and sorted sets. The right pattern depends on whether you need top-down traversal, bottom-up lookup, or path queries.

## Pattern 1: Parent Reference (Adjacency List)

Each node stores its parent ID - the simplest approach:

```bash
# Category tree
# Root: category:1 (Electronics)
#   - category:2 (Laptops)
#   - category:3 (Phones)
#     - category:4 (iPhones)

HSET category:1 name "Electronics" parent ""
HSET category:2 name "Laptops"     parent "1"
HSET category:3 name "Phones"      parent "1"
HSET category:4 name "iPhones"     parent "3"
```

Getting a node's parent:

```bash
HGET category:4 parent  # "3"
HGET category:3 name    # "Phones"
```

Finding all children requires a scan - this pattern is best for small trees or when you primarily navigate up (child to root).

## Pattern 2: Children Sets (Materialized Children)

Store each node's children explicitly for fast top-down traversal:

```python
import redis

r = redis.Redis(decode_responses=True)

def add_node(r, node_id, name, parent_id=None):
    pipe = r.pipeline()
    parent_str = str(parent_id) if parent_id is not None else ""
    pipe.hset(f"node:{node_id}", mapping={
        "name": name,
        "parent": parent_str
    })
    if parent_id is not None:
        pipe.sadd(f"node:{parent_id}:children", str(node_id))
    pipe.execute()

def get_children(r, node_id):
    child_ids = r.smembers(f"node:{node_id}:children")
    pipe = r.pipeline(transaction=False)
    for cid in child_ids:
        pipe.hgetall(f"node:{cid}")
    return pipe.execute()

def get_subtree(r, node_id):
    """Breadth-first traversal of a subtree."""
    result = []
    queue = [str(node_id)]
    while queue:
        current = queue.pop(0)
        node = r.hgetall(f"node:{current}")
        if node:
            result.append(node)
        children = r.smembers(f"node:{current}:children")
        queue.extend(children)
    return result

add_node(r, 1, "Electronics")
add_node(r, 2, "Laptops", parent_id=1)
add_node(r, 3, "Phones", parent_id=1)
add_node(r, 4, "iPhones", parent_id=3)

print(get_children(r, 1))   # Laptops and Phones
print(get_subtree(r, 1))    # All nodes under Electronics
```

## Pattern 3: Materialized Path

Store the full path for each node - enables efficient ancestor and descendant queries:

```bash
# Each node stores its path from root
HSET category:1 name "Electronics"  path "/1"
HSET category:2 name "Laptops"      path "/1/2"
HSET category:3 name "Phones"       path "/1/3"
HSET category:4 name "iPhones"      path "/1/3/4"
```

Finding all descendants of node 3 (Phones):

```python
def find_descendants(r, node_id, all_node_ids):
    """Find all nodes whose path starts with the target path."""
    target_path = r.hget(f"category:{node_id}", "path")
    descendants = []
    for nid in all_node_ids:
        path = r.hget(f"category:{nid}", "path")
        if path and path.startswith(target_path + "/"):
            descendants.append(nid)
    return descendants
```

For large trees, store paths in a sorted set for efficient prefix queries:

```python
def index_path(r, node_id, path):
    r.zadd("category:paths", {f"{path}:{node_id}": 0})

def find_descendants_fast(r, node_path):
    """Use ZRANGEBYLEX for prefix search."""
    results = r.zrangebylex(
        "category:paths",
        f"[{node_path}/",
        f"[{node_path}/\xff"
    )
    return [item.split(":")[-1] for item in results]
```

## Pattern 4: Depth-First Path for Breadcrumbs

Store the full ancestor chain for breadcrumb generation:

```python
def get_breadcrumb(r, node_id):
    """Walk up the tree using parent references."""
    breadcrumb = []
    current = str(node_id)
    while current:
        node = r.hgetall(f"node:{current}")
        if not node:
            break
        breadcrumb.append(node["name"])
        current = node.get("parent", "")
    return list(reversed(breadcrumb))

print(get_breadcrumb(r, 4))
# ["Electronics", "Phones", "iPhones"]
```

## Choosing the Right Pattern

```text
Pattern              Best for                   Traversal
Parent reference     Bottom-up only             Up only
Children sets        Top-down traversal          Down (BFS/DFS)
Materialized path    Ancestor/descendant queries Both directions
Both sets + parent   Full bidirectional          Both directions
```

For org charts, file systems, and product categories, the combination of children sets + parent reference gives the best query flexibility at the cost of maintaining both when modifying the tree.

## Deleting a Subtree

```python
def delete_subtree(r, node_id):
    queue = [str(node_id)]
    to_delete = []
    while queue:
        current = queue.pop(0)
        children = r.smembers(f"node:{current}:children")
        queue.extend(children)
        to_delete.append(current)

    pipe = r.pipeline()
    for nid in to_delete:
        pipe.delete(f"node:{nid}")
        pipe.delete(f"node:{nid}:children")
    parent_id = r.hget(f"node:{node_id}", "parent")
    if parent_id:
        pipe.srem(f"node:{parent_id}:children", str(node_id))
    pipe.execute()
```

## Summary

Model tree data in Redis with parent references for upward navigation, children sets for downward traversal, and materialized paths for ancestor/descendant queries. Use all three together for fully bidirectional tree operations. Always update parent-child relationships atomically using MULTI/EXEC or pipelining to prevent dangling references.
