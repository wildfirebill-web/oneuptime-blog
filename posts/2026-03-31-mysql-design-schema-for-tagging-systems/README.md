# How to Design a Schema for Tagging Systems in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Tagging, Schema Design, Many-to-Many

Description: Learn how to build a flexible tagging system in MySQL with normalized tag tables, junction tables, and efficient search queries.

---

A tagging system lets users categorize content with free-form labels. The normalized approach stores tags separately and links them to content through a junction table.

## Core Schema

```sql
CREATE TABLE posts (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    title      VARCHAR(255) NOT NULL,
    body       TEXT         NOT NULL,
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
);

CREATE TABLE tags (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name       VARCHAR(100) NOT NULL,
    slug       VARCHAR(100) NOT NULL,
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_slug (slug)
);

CREATE TABLE post_tags (
    post_id INT UNSIGNED NOT NULL,
    tag_id  INT UNSIGNED NOT NULL,
    PRIMARY KEY (post_id, tag_id),
    KEY idx_tag_post (tag_id, post_id),
    CONSTRAINT fk_pt_post FOREIGN KEY (post_id) REFERENCES posts (id) ON DELETE CASCADE,
    CONSTRAINT fk_pt_tag  FOREIGN KEY (tag_id)  REFERENCES tags  (id) ON DELETE CASCADE
);
```

## Inserting Tags

Use `INSERT IGNORE` when the tag may already exist:

```sql
INSERT IGNORE INTO tags (name, slug)
VALUES ('mysql', 'mysql'), ('schema-design', 'schema-design');

-- Associate tags with a post
INSERT INTO post_tags (post_id, tag_id)
SELECT 1, id FROM tags WHERE slug IN ('mysql', 'schema-design');
```

## Finding Posts by Tag

```sql
SELECT p.id, p.title
FROM   posts p
JOIN   post_tags pt ON pt.post_id = p.id
JOIN   tags t       ON t.id = pt.tag_id
WHERE  t.slug = 'mysql'
ORDER BY p.created_at DESC;
```

## Finding Posts With Multiple Tags (AND)

```sql
SELECT p.id, p.title
FROM   posts p
JOIN   post_tags pt ON pt.post_id = p.id
JOIN   tags t       ON t.id = pt.tag_id
WHERE  t.slug IN ('mysql', 'schema-design')
GROUP BY p.id
HAVING COUNT(DISTINCT t.id) = 2;
```

## Getting Tags for a Post

```sql
SELECT t.name, t.slug
FROM   tags t
JOIN   post_tags pt ON pt.tag_id = t.id
WHERE  pt.post_id = 1;
```

## Popular Tags

```sql
SELECT t.name, COUNT(pt.post_id) AS usage_count
FROM   tags t
JOIN   post_tags pt ON pt.tag_id = t.id
GROUP BY t.id, t.name
ORDER BY usage_count DESC
LIMIT 20;
```

## Tag Cloud With Weighted Frequency

```sql
SELECT
    t.name,
    COUNT(pt.post_id) AS cnt,
    ROUND(COUNT(pt.post_id) * 100.0 /
          (SELECT COUNT(*) FROM post_tags), 2) AS pct
FROM   tags t
LEFT JOIN post_tags pt ON pt.tag_id = t.id
GROUP BY t.id, t.name
ORDER BY cnt DESC;
```

## Removing a Tag From a Post

```sql
DELETE FROM post_tags
WHERE  post_id = 1
AND    tag_id  = (SELECT id FROM tags WHERE slug = 'mysql');
```

## Summary

Build tagging systems with a normalized `tags` table and a junction `post_tags` table. Index `(tag_id, post_id)` for reverse lookups. Use `COUNT(DISTINCT tag_id) = N` in `HAVING` to find posts matching multiple tags simultaneously. Slugs on tags enable URL-friendly filtering without case-sensitivity concerns.
