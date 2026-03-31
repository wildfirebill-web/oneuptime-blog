# How to Create Junction Tables for Many-to-Many Relationships in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Junction Table, Schema Design, Relationship

Description: A step-by-step guide to creating junction tables in MySQL for many-to-many relationships, covering keys, indexes, and cascade rules.

---

A junction table (also called a bridge or associative table) sits between two entities that have a many-to-many relationship. Creating it correctly is essential for query performance and referential integrity.

## Anatomy of a Junction Table

A well-designed junction table has:
- Foreign keys referencing both parent tables
- A composite primary key on the two foreign key columns
- A supplementary index on the second column for reverse lookups
- Optional attribute columns describing the relationship

## Step-by-Step Example

Suppose you have `tags` and `articles`:

```sql
CREATE TABLE articles (
    id      INT UNSIGNED NOT NULL AUTO_INCREMENT,
    title   VARCHAR(255) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE tags (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_name (name)
);
```

Now create the junction table:

```sql
CREATE TABLE article_tags (
    article_id INT UNSIGNED NOT NULL,
    tag_id     INT UNSIGNED NOT NULL,
    tagged_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (article_id, tag_id),
    KEY idx_tag_article (tag_id, article_id),
    CONSTRAINT fk_at_article
        FOREIGN KEY (article_id) REFERENCES articles (id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    CONSTRAINT fk_at_tag
        FOREIGN KEY (tag_id)     REFERENCES tags (id)
        ON DELETE CASCADE ON UPDATE CASCADE
);
```

The second index `(tag_id, article_id)` is a covering index for queries that look up articles by tag.

## Inserting Associations

```sql
-- Tag an article with multiple tags
INSERT INTO article_tags (article_id, tag_id) VALUES
    (1, 5),
    (1, 8),
    (1, 12);
```

## Querying Through the Junction

```sql
-- All tags for a given article
SELECT t.name
FROM   tags t
JOIN   article_tags at ON at.tag_id = t.id
WHERE  at.article_id = 1;

-- All articles for a given tag
SELECT a.title, at.tagged_at
FROM   articles a
JOIN   article_tags at ON at.article_id = a.id
WHERE  at.tag_id = 5;
```

## Removing an Association

```sql
DELETE FROM article_tags
WHERE  article_id = 1
AND    tag_id = 8;
```

Because `ON DELETE CASCADE` is set on the parent tables, deleting an article or a tag automatically removes all rows in `article_tags`.

## Using an Auto-Increment Surrogate Key

Sometimes you need a single-column primary key on the junction table (for example, to reference from another table):

```sql
CREATE TABLE article_tags (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    article_id INT UNSIGNED NOT NULL,
    tag_id     INT UNSIGNED NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_article_tag (article_id, tag_id),
    KEY idx_tag (tag_id),
    CONSTRAINT fk_at_article FOREIGN KEY (article_id) REFERENCES articles (id) ON DELETE CASCADE,
    CONSTRAINT fk_at_tag     FOREIGN KEY (tag_id)     REFERENCES tags (id)     ON DELETE CASCADE
);
```

The `UNIQUE KEY` still prevents duplicates while the surrogate `id` serves as a stable reference point.

## Summary

Create junction tables with a composite primary key on the two foreign key columns, a reverse index on the second column, and `ON DELETE CASCADE` on both constraints. Add extra columns to capture relationship metadata. If another table needs to reference the junction row, add a surrogate auto-increment key alongside a unique constraint on the pair.
