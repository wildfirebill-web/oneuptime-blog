# How to Implement Closure Table for Hierarchical Data in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Hierarchy, Closure Table, Query, Schema

Description: Learn how to implement the closure table pattern in MySQL to efficiently query hierarchical data like categories, org charts, and threaded comments.

---

## What Is a Closure Table?

A closure table is a relational pattern for storing hierarchical data. Instead of putting a `parent_id` column in a single table, you create a separate table that stores every ancestor-descendant pair in the tree - including a node's relationship to itself. This makes queries for subtrees and paths trivially fast using standard JOINs.

## Schema Setup

Start with a `categories` table for the nodes and a `category_closure` table for the relationships.

```sql
CREATE TABLE categories (
  id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL
);

CREATE TABLE category_closure (
  ancestor   INT UNSIGNED NOT NULL,
  descendant INT UNSIGNED NOT NULL,
  depth      INT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (ancestor, descendant),
  INDEX idx_descendant (descendant),
  FOREIGN KEY (ancestor)   REFERENCES categories(id) ON DELETE CASCADE,
  FOREIGN KEY (descendant) REFERENCES categories(id) ON DELETE CASCADE
);
```

The `depth` column records how many levels separate ancestor from descendant.

## Inserting a New Node

When inserting a new category under a parent, copy all ancestor rows of the parent and add a new row for the node itself.

```sql
-- Insert the node
INSERT INTO categories (name) VALUES ('Electronics');
SET @new_id = LAST_INSERT_ID();

-- Insert the self-reference row
INSERT INTO category_closure (ancestor, descendant, depth)
VALUES (@new_id, @new_id, 0);

-- Insert rows linking all existing ancestors of the parent
INSERT INTO category_closure (ancestor, descendant, depth)
SELECT ancestor, @new_id, depth + 1
FROM category_closure
WHERE descendant = @parent_id;
```

## Querying Descendants

Retrieve all descendants of a node at any depth:

```sql
SELECT c.*
FROM categories c
JOIN category_closure cc ON cc.descendant = c.id
WHERE cc.ancestor = 1
  AND cc.depth > 0;
```

To limit to direct children only, add `AND cc.depth = 1`.

## Querying Ancestors (Breadcrumb Path)

```sql
SELECT c.*, cc.depth
FROM categories c
JOIN category_closure cc ON cc.ancestor = c.id
WHERE cc.descendant = 5
ORDER BY cc.depth DESC;
```

## Deleting a Subtree

Because the closure table uses `ON DELETE CASCADE` on the `categories` table, deleting a node removes its closure rows. To delete just a subtree, delete the descendants first:

```sql
DELETE FROM categories
WHERE id IN (
  SELECT descendant
  FROM category_closure
  WHERE ancestor = 3 AND depth > 0
);
DELETE FROM categories WHERE id = 3;
```

## Moving a Subtree

Moving is the most complex operation. Disconnect the subtree from its old ancestors, then re-attach under the new parent:

```sql
-- Remove old ancestor links above the moved subtree
DELETE FROM category_closure
WHERE descendant IN (
    SELECT descendant FROM (
      SELECT descendant FROM category_closure WHERE ancestor = @moved_id
    ) AS sub
  )
  AND ancestor NOT IN (
    SELECT descendant FROM (
      SELECT descendant FROM category_closure WHERE ancestor = @moved_id
    ) AS sub2
  );

-- Re-attach under the new parent
INSERT INTO category_closure (ancestor, descendant, depth)
SELECT p.ancestor, c.descendant, p.depth + c.depth + 1
FROM category_closure p
JOIN category_closure c ON c.ancestor = @moved_id
WHERE p.descendant = @new_parent_id;
```

## Summary

The closure table pattern trades write complexity for read performance. Every subtree or ancestor query is a single JOIN rather than a recursive CTE or procedural loop. It is the right choice when hierarchies are read far more often than they are modified, such as product category trees or threaded comment systems. Use the `depth` column to filter by level and to reconstruct ordered breadcrumb paths.
