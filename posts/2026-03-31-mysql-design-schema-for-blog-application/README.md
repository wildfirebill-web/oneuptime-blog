# How to Design a Schema for a Blog Application in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Blog, Schema Design, Application

Description: Learn how to design a complete blog application schema in MySQL covering users, posts, categories, tags, comments, and indexes.

---

A blog schema must handle users, authored posts, categorization, tagging, and threaded comments. This guide walks through each entity and the relationships between them.

## Users

```sql
CREATE TABLE users (
    id            INT UNSIGNED NOT NULL AUTO_INCREMENT,
    username      VARCHAR(50)  NOT NULL,
    email         VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    bio           TEXT         NULL,
    created_at    DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_username (username),
    UNIQUE KEY uq_email (email)
);
```

## Categories

```sql
CREATE TABLE categories (
    id        INT UNSIGNED NOT NULL AUTO_INCREMENT,
    parent_id INT UNSIGNED NULL,
    name      VARCHAR(100) NOT NULL,
    slug      VARCHAR(100) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_slug (slug),
    KEY idx_parent (parent_id),
    CONSTRAINT fk_cat_parent FOREIGN KEY (parent_id) REFERENCES categories (id)
);
```

## Posts

```sql
CREATE TABLE posts (
    id           INT UNSIGNED NOT NULL AUTO_INCREMENT,
    author_id    INT UNSIGNED NOT NULL,
    category_id  INT UNSIGNED NULL,
    title        VARCHAR(255) NOT NULL,
    slug         VARCHAR(255) NOT NULL,
    body         LONGTEXT     NOT NULL,
    status       ENUM('draft','published','archived') NOT NULL DEFAULT 'draft',
    published_at DATETIME     NULL,
    created_at   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_slug (slug),
    KEY idx_author    (author_id),
    KEY idx_category  (category_id),
    KEY idx_status_pub (status, published_at),
    FULLTEXT KEY ft_title_body (title, body),
    CONSTRAINT fk_post_author   FOREIGN KEY (author_id)   REFERENCES users      (id),
    CONSTRAINT fk_post_category FOREIGN KEY (category_id) REFERENCES categories (id) ON DELETE SET NULL
);
```

## Tags and Post-Tag Junction

```sql
CREATE TABLE tags (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uq_slug (slug)
);

CREATE TABLE post_tags (
    post_id INT UNSIGNED NOT NULL,
    tag_id  INT UNSIGNED NOT NULL,
    PRIMARY KEY (post_id, tag_id),
    KEY idx_tag (tag_id),
    CONSTRAINT fk_pt_post FOREIGN KEY (post_id) REFERENCES posts (id) ON DELETE CASCADE,
    CONSTRAINT fk_pt_tag  FOREIGN KEY (tag_id)  REFERENCES tags  (id) ON DELETE CASCADE
);
```

## Comments

```sql
CREATE TABLE comments (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    post_id    INT UNSIGNED NOT NULL,
    author_id  INT UNSIGNED NULL,
    parent_id  INT UNSIGNED NULL,
    body       TEXT         NOT NULL,
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_post   (post_id),
    KEY idx_parent (parent_id),
    CONSTRAINT fk_comment_post   FOREIGN KEY (post_id)   REFERENCES posts    (id) ON DELETE CASCADE,
    CONSTRAINT fk_comment_author FOREIGN KEY (author_id) REFERENCES users    (id) ON DELETE SET NULL,
    CONSTRAINT fk_comment_parent FOREIGN KEY (parent_id) REFERENCES comments (id) ON DELETE CASCADE
);
```

## Querying Published Posts With Tag Count

```sql
SELECT p.title, u.username, COUNT(pt.tag_id) AS tag_count
FROM   posts p
JOIN   users u      ON u.id = p.author_id
LEFT JOIN post_tags pt ON pt.post_id = p.id
WHERE  p.status = 'published'
GROUP BY p.id
ORDER BY p.published_at DESC
LIMIT 10;
```

## Full-Text Search

```sql
SELECT id, title, MATCH(title, body) AGAINST ('mysql schema' IN NATURAL LANGUAGE MODE) AS score
FROM   posts
WHERE  status = 'published'
  AND  MATCH(title, body) AGAINST ('mysql schema' IN NATURAL LANGUAGE MODE)
ORDER BY score DESC;
```

## Summary

A blog schema centers on a `posts` table linked to `users`, `categories`, and `tags`. Use a self-referencing `categories` table for nested categories and a self-referencing `comments` table for threading. Add a FULLTEXT index on `title` and `body` for search. Index `(status, published_at)` to support efficient listing of published posts.
