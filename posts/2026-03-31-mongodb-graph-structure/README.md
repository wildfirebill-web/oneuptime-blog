# How to Implement a Graph Structure in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Graph, Data Structure, Relationship, Aggregation

Description: Learn how to model and traverse graph structures in MongoDB using adjacency lists and the $graphLookup aggregation stage.

---

## Graph Modeling in MongoDB

MongoDB does not have native graph primitives, but graph data maps naturally to documents. Two main approaches exist:

1. **Adjacency list** - each node stores a list of neighbors or a parent reference
2. **Edge collection** - a separate collection stores edges with `from` and `to` references

## Adjacency List Model

Each node document stores its parent and/or children:

```javascript
// Nodes collection
{ "_id": "CEO", "name": "Alice", "reportsTo": null, "title": "Chief Executive Officer" }
{ "_id": "CTO", "name": "Bob",   "reportsTo": "CEO", "title": "Chief Technology Officer" }
{ "_id": "ENG1", "name": "Carol", "reportsTo": "CTO", "title": "Senior Engineer" }
{ "_id": "ENG2", "name": "Dave",  "reportsTo": "CTO", "title": "Engineer" }
{ "_id": "PM1",  "name": "Eve",   "reportsTo": "CEO", "title": "Product Manager" }
```

## Traversing with $graphLookup

`$graphLookup` recursively follows references to traverse the graph:

```javascript
// Find all reports under "CTO" (any depth)
db.employees.aggregate([
  { $match: { _id: "CTO" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "reportsTo",
      as: "subordinates",
      maxDepth: 10,
      depthField: "depth"
    }
  }
])
```

## Finding the Ancestry Path

To find all ancestors of a node (traverse upward):

```javascript
db.employees.aggregate([
  { $match: { _id: "ENG1" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$reportsTo",
      connectFromField: "reportsTo",
      connectToField: "_id",
      as: "ancestors",
      maxDepth: 10,
      depthField: "level"
    }
  }
])
```

## Edge Collection Model

For general graphs (not trees), use a separate edges collection:

```javascript
// nodes collection
{ "_id": "A", "label": "Page A", "type": "page" }
{ "_id": "B", "label": "Page B", "type": "page" }

// edges collection
{ "_id": "e1", "from": "A", "to": "B", "weight": 1.0, "type": "link" }
{ "_id": "e2", "from": "B", "to": "C", "weight": 0.8, "type": "link" }
```

## Querying Direct Neighbors

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client.myDatabase

def get_neighbors(node_id: str) -> list:
    edge_ids = [
        e["to"]
        for e in db.edges.find({"from": node_id}, {"to": 1})
    ]
    return list(db.nodes.find({"_id": {"$in": edge_ids}}))
```

## BFS Traversal in Python

```python
from collections import deque

def bfs(start_id: str, max_depth: int = 5) -> dict:
    visited = {}
    queue = deque([(start_id, 0)])

    while queue:
        node_id, depth = queue.popleft()
        if node_id in visited or depth > max_depth:
            continue
        visited[node_id] = depth

        neighbors = [e["to"] for e in db.edges.find({"from": node_id})]
        for neighbor in neighbors:
            if neighbor not in visited:
                queue.append((neighbor, depth + 1))

    return visited
```

## Detecting Cycles

Use a DFS with a visited set and a current-path set:

```python
def has_cycle(start_id: str) -> bool:
    visited = set()
    path = set()

    def dfs(node_id):
        visited.add(node_id)
        path.add(node_id)
        for edge in db.edges.find({"from": node_id}):
            neighbor = edge["to"]
            if neighbor not in visited:
                if dfs(neighbor):
                    return True
            elif neighbor in path:
                return True
        path.discard(node_id)
        return False

    return dfs(start_id)
```

## Indexing for Graph Queries

```javascript
db.edges.createIndex({ "from": 1 })
db.edges.createIndex({ "to": 1 })
db.nodes.createIndex({ "type": 1 })
```

These indexes make neighbor lookups and `$graphLookup` starting conditions efficient.

## Summary

MongoDB graph structures use either an adjacency list (parent reference per node) or a separate edges collection for general graphs. The `$graphLookup` aggregation stage handles recursive traversal within the database. For BFS, DFS, or cycle detection, implement the algorithm in application code using edge queries backed by indexes on `from` and `to` fields.
