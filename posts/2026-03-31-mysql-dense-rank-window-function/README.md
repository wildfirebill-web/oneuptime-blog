# How to Use DENSE_RANK() Window Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Function, Ranking

Description: Learn how to use MySQL's DENSE_RANK() window function to assign consecutive ranks without gaps, with examples comparing it to RANK() and ROW_NUMBER().

---

## What Is DENSE_RANK()?

`DENSE_RANK()` is a window function introduced in MySQL 8.0 that assigns a rank to each row within a partition, ordered by a specified column. Unlike `RANK()`, `DENSE_RANK()` does not skip rank numbers after ties - it assigns consecutive ranks.

| Function | Handles ties | Skips ranks |
|---|---|---|
| `ROW_NUMBER()` | Each tie gets unique number | Never |
| `RANK()` | Ties share same rank | Yes - gaps appear |
| `DENSE_RANK()` | Ties share same rank | No - consecutive |

## Basic Syntax

```sql
DENSE_RANK() OVER (
  [PARTITION BY partition_expr]
  ORDER BY sort_expr [ASC|DESC]
)
```

## Example Setup

```sql
CREATE TABLE scores (
  student VARCHAR(50),
  subject VARCHAR(50),
  score INT
);

INSERT INTO scores VALUES
  ('Alice', 'Math', 95),
  ('Bob', 'Math', 87),
  ('Carol', 'Math', 87),
  ('Dave', 'Math', 75),
  ('Eve', 'Math', 75),
  ('Frank', 'Math', 60);
```

## DENSE_RANK() vs RANK() Side by Side

```sql
SELECT
  student,
  score,
  RANK() OVER (ORDER BY score DESC) AS rank_with_gaps,
  DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank
FROM scores;
```

Result:

```text
+---------+-------+----------------+------------+
| student | score | rank_with_gaps | dense_rank |
+---------+-------+----------------+------------+
| Alice   |    95 |              1 |          1 |
| Bob     |    87 |              2 |          2 |
| Carol   |    87 |              2 |          2 |
| Dave    |    75 |              4 |          3 |
| Eve     |    75 |              4 |          3 |
| Frank   |    60 |              6 |          4 |
+---------+-------+----------------+------------+
```

With `RANK()`, ranks 3 and 5 are skipped after the two-way ties. With `DENSE_RANK()`, ranks are consecutive: 1, 2, 2, 3, 3, 4.

## Using PARTITION BY

Rank students within each subject:

```sql
INSERT INTO scores VALUES
  ('Alice', 'English', 88),
  ('Bob', 'English', 88),
  ('Carol', 'English', 92),
  ('Dave', 'English', 70);

SELECT
  student,
  subject,
  score,
  DENSE_RANK() OVER (
    PARTITION BY subject
    ORDER BY score DESC
  ) AS subject_rank
FROM scores
ORDER BY subject, subject_rank;
```

## Filtering to Top N per Group

Use `DENSE_RANK()` when you want all students who share the top rank:

```sql
SELECT student, subject, score, dr
FROM (
  SELECT *,
    DENSE_RANK() OVER (
      PARTITION BY subject
      ORDER BY score DESC
    ) AS dr
  FROM scores
) ranked
WHERE dr <= 2;  -- Top 2 distinct score tiers per subject
```

With `ROW_NUMBER()`, ties in the top 2 would exclude some students. `DENSE_RANK()` includes all tied students at each rank level.

## Percentile Groups with DENSE_RANK()

Assign students to grade tiers based on their rank:

```sql
SELECT
  student,
  score,
  DENSE_RANK() OVER (ORDER BY score DESC) AS rank,
  CASE DENSE_RANK() OVER (ORDER BY score DESC)
    WHEN 1 THEN 'Gold'
    WHEN 2 THEN 'Silver'
    WHEN 3 THEN 'Bronze'
    ELSE 'Participant'
  END AS award
FROM scores
WHERE subject = 'Math';
```

## Ranking in ORDER BY

```sql
-- Sort by dense rank and then alphabetically for ties
SELECT
  student,
  score,
  DENSE_RANK() OVER (ORDER BY score DESC) AS dr
FROM scores
WHERE subject = 'Math'
ORDER BY dr, student;
```

## Summary

`DENSE_RANK()` assigns consecutive rank numbers to ordered rows, sharing ranks among ties without creating gaps. Use it instead of `RANK()` when rank gaps would be confusing (such as competition standings) or when filtering for the top N distinct score tiers per group.
