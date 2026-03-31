# How to Design Polymorphic Associations in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Polymorphic Association, Schema Design, Relationship

Description: Learn how to implement polymorphic associations in MySQL for comments, tags, or likes that belong to multiple entity types.

---

A polymorphic association lets a single child table relate to multiple parent tables. The classic example is a `comments` table where comments can belong to a `post`, a `photo`, or a `video` - all stored in one table.

## The Problem

You want to avoid creating separate `post_comments`, `photo_comments`, `video_comments` tables. A polymorphic approach uses a `type` discriminator column alongside the foreign key ID.

## Schema Design

```sql
CREATE TABLE posts (
    id    INT UNSIGNED NOT NULL AUTO_INCREMENT,
    title VARCHAR(255) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE photos (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT,
    filename VARCHAR(255) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE comments (
    id              INT UNSIGNED NOT NULL AUTO_INCREMENT,
    commentable_id   INT UNSIGNED NOT NULL,
    commentable_type ENUM('post', 'photo') NOT NULL,
    body             TEXT         NOT NULL,
    created_at       DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_polymorphic (commentable_type, commentable_id)
);
```

Note: no traditional foreign key is used because MySQL foreign keys cannot reference multiple tables conditionally. Referential integrity is enforced in the application or via triggers.

## Inserting Comments

```sql
-- Comment on a post
INSERT INTO comments (commentable_id, commentable_type, body)
VALUES (1, 'post', 'Great article!');

-- Comment on a photo
INSERT INTO comments (commentable_id, commentable_type, body)
VALUES (5, 'photo', 'Nice shot!');
```

## Querying Comments by Type

```sql
-- All comments for post #1
SELECT c.body, c.created_at
FROM   comments c
WHERE  c.commentable_type = 'post'
AND    c.commentable_id   = 1;
```

## Joining Back to the Parent

```sql
-- Comments with post title
SELECT c.body, p.title AS post_title
FROM   comments c
JOIN   posts p ON p.id = c.commentable_id
                AND c.commentable_type = 'post'
ORDER BY c.created_at DESC;
```

## Enforcing Referential Integrity With a Trigger

```sql
DELIMITER $$
CREATE TRIGGER trg_comment_fk BEFORE INSERT ON comments
FOR EACH ROW
BEGIN
    IF NEW.commentable_type = 'post' THEN
        IF NOT EXISTS (SELECT 1 FROM posts WHERE id = NEW.commentable_id) THEN
            SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Invalid post ID';
        END IF;
    ELSEIF NEW.commentable_type = 'photo' THEN
        IF NOT EXISTS (SELECT 1 FROM photos WHERE id = NEW.commentable_id) THEN
            SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Invalid photo ID';
        END IF;
    END IF;
END$$
DELIMITER ;
```

## Alternative: Separate Nullable Foreign Keys

For better integrity at the cost of extra nullable columns:

```sql
CREATE TABLE comments (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT,
    post_id  INT UNSIGNED NULL,
    photo_id INT UNSIGNED NULL,
    body     TEXT NOT NULL,
    PRIMARY KEY (id),
    CONSTRAINT fk_comment_post  FOREIGN KEY (post_id)  REFERENCES posts (id)  ON DELETE CASCADE,
    CONSTRAINT fk_comment_photo FOREIGN KEY (photo_id) REFERENCES photos (id) ON DELETE CASCADE,
    CONSTRAINT chk_one_parent CHECK (
        (post_id IS NOT NULL) + (photo_id IS NOT NULL) = 1
    )
);
```

## Summary

Polymorphic associations use a `type` discriminator column alongside a shared ID column. Index both columns together for query efficiency. Because native foreign keys cannot span multiple tables, enforce integrity through application logic, triggers, or by using the separate nullable foreign key pattern with a CHECK constraint.
