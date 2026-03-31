# How to Implement a Linked List Structure in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Linked List, Data Structure, Pattern, Document

Description: Learn how to implement a singly and doubly linked list in MongoDB using document references for ordered, insertable, and traversable sequences.

---

## When to Use a Linked List in MongoDB?

MongoDB arrays handle most ordered sequence needs natively. A linked list is useful when you need:
- O(1) insertions at arbitrary positions without shifting large arrays
- Sequences too long to store as a single array (arrays > a few thousand elements in one document)
- Ordered traversal with dynamic insertion/deletion at scale

## Singly Linked List Schema

Each node is a document with a pointer to the next node:

```javascript
// Node document
{
  "_id": "node-1",
  "listId": "playlist-123",
  "value": { "songId": "song-a", "title": "Song A" },
  "nextId": "node-2",
  "createdAt": ISODate("2026-03-31T09:00:00Z")
}

// Head document (tracks the list head)
{
  "_id": "playlist-123",
  "headId": "node-1",
  "size": 3
}
```

## Inserting at the Head

```python
from pymongo import MongoClient
import uuid

client = MongoClient("mongodb://localhost:27017/")
db = client.myDatabase

def prepend(list_id: str, value: dict) -> str:
    node_id = str(uuid.uuid4())

    # Get current head
    lst = db.lists.find_one({"_id": list_id})
    current_head = lst["headId"] if lst else None

    # Insert the new node
    db.nodes.insert_one({
        "_id": node_id,
        "listId": list_id,
        "value": value,
        "nextId": current_head
    })

    # Update list head
    db.lists.update_one(
        {"_id": list_id},
        {"$set": {"headId": node_id}, "$inc": {"size": 1}},
        upsert=True
    )
    return node_id
```

## Inserting After a Node

```python
def insert_after(list_id: str, after_id: str, value: dict) -> str:
    node_id = str(uuid.uuid4())

    # Get the node we are inserting after
    after_node = db.nodes.find_one({"_id": after_id, "listId": list_id})
    if not after_node:
        raise ValueError(f"Node {after_id} not found")

    # Insert new node pointing to after_node's next
    db.nodes.insert_one({
        "_id": node_id,
        "listId": list_id,
        "value": value,
        "nextId": after_node.get("nextId")
    })

    # Update after_node to point to new node
    db.nodes.update_one(
        {"_id": after_id},
        {"$set": {"nextId": node_id}}
    )

    db.lists.update_one({"_id": list_id}, {"$inc": {"size": 1}})
    return node_id
```

## Traversing the List

```python
def traverse(list_id: str) -> list:
    lst = db.lists.find_one({"_id": list_id})
    if not lst or not lst.get("headId"):
        return []

    items = []
    current_id = lst["headId"]
    visited = set()

    while current_id and current_id not in visited:
        visited.add(current_id)
        node = db.nodes.find_one({"_id": current_id})
        if not node:
            break
        items.append(node["value"])
        current_id = node.get("nextId")

    return items
```

## Removing a Node

```python
def remove(list_id: str, node_id: str) -> bool:
    # Find the node before the one to remove
    prev = db.nodes.find_one({"listId": list_id, "nextId": node_id})
    target = db.nodes.find_one({"_id": node_id, "listId": list_id})

    if not target:
        return False

    next_id = target.get("nextId")

    if prev:
        db.nodes.update_one({"_id": prev["_id"]}, {"$set": {"nextId": next_id}})
    else:
        # Removing the head
        db.lists.update_one({"_id": list_id}, {"$set": {"headId": next_id}})

    db.nodes.delete_one({"_id": node_id})
    db.lists.update_one({"_id": list_id}, {"$inc": {"size": -1}})
    return True
```

## Doubly Linked List

Add a `prevId` field to support reverse traversal:

```javascript
{
  "_id": "node-2",
  "listId": "playlist-123",
  "value": { "songId": "song-b" },
  "nextId": "node-3",
  "prevId": "node-1"
}
```

Update both `prevId` and `nextId` on insert and remove operations.

## Indexing

```javascript
db.nodes.createIndex({ "listId": 1, "nextId": 1 })
```

This speeds up "find previous node" queries needed during removal.

## Summary

MongoDB linked lists use a `nodes` collection with `nextId` references and a separate `lists` document tracking the head pointer. Insertions at arbitrary positions require only updating two nodes rather than shifting an entire array. Use this pattern for very long ordered sequences where array-in-document approaches hit the 16MB BSON document size limit.
