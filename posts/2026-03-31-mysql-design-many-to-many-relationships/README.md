# How to Design Many-to-Many Relationships in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Relationship, Schema Design, Junction Table

Description: Learn how to model many-to-many relationships in MySQL using junction tables, composite keys, and efficient join strategies.

---

A many-to-many relationship means rows in table A can relate to multiple rows in table B and vice versa. MySQL cannot express this directly - you need a junction table (also called an associative or bridge table) that holds the pairing.

## The Problem

Consider students and courses: a student can enroll in many courses, and a course can have many students. You cannot place the foreign key on either side alone without violating normalization.

## Schema With a Junction Table

```sql
CREATE TABLE students (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(150) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE courses (
    id    INT UNSIGNED NOT NULL AUTO_INCREMENT,
    title VARCHAR(200) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE enrollments (
    student_id INT UNSIGNED NOT NULL,
    course_id  INT UNSIGNED NOT NULL,
    enrolled_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (student_id, course_id),
    KEY idx_course (course_id),
    CONSTRAINT fk_enroll_student
        FOREIGN KEY (student_id) REFERENCES students (id)
        ON DELETE CASCADE,
    CONSTRAINT fk_enroll_course
        FOREIGN KEY (course_id) REFERENCES courses (id)
        ON DELETE CASCADE
);
```

The composite primary key `(student_id, course_id)` prevents duplicate enrollments. The extra index on `course_id` supports queries that filter by course.

## Querying the Relationship

```sql
-- All courses for a student
SELECT c.title, e.enrolled_at
FROM   courses c
JOIN   enrollments e ON e.course_id = c.id
WHERE  e.student_id = 7;
```

```sql
-- All students in a specific course
SELECT s.name, e.enrolled_at
FROM   students s
JOIN   enrollments e ON e.student_id = s.id
WHERE  e.course_id = 3;
```

## Adding Attributes to the Relationship

The junction table can carry extra columns that describe the relationship itself:

```sql
ALTER TABLE enrollments
    ADD COLUMN grade      CHAR(2) NULL,
    ADD COLUMN completed  TINYINT(1) NOT NULL DEFAULT 0;
```

This makes `enrollments` a richer entity rather than just a pairing, but it is still the correct pattern.

## Preventing Duplicates at Insert

With the composite primary key in place, a duplicate insert raises an error. Use `INSERT IGNORE` or `INSERT ... ON DUPLICATE KEY UPDATE` to handle it gracefully:

```sql
INSERT INTO enrollments (student_id, course_id)
VALUES (7, 3)
ON DUPLICATE KEY UPDATE enrolled_at = enrolled_at;
```

## Finding Students Not Enrolled in a Course

```sql
SELECT s.id, s.name
FROM   students s
WHERE  s.id NOT IN (
    SELECT student_id FROM enrollments WHERE course_id = 3
);
```

## Summary

Many-to-many relationships require a junction table with foreign keys pointing to both parent tables. A composite primary key on the junction table prevents duplicates. Extra columns on the junction table represent attributes of the relationship itself. Index the second column of the composite key to support reverse lookups efficiently.
