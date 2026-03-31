# How to Apply Boyce-Codd Normal Form (BCNF) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Normalization, BCNF, Schema Design, Functional Dependency

Description: Learn what Boyce-Codd Normal Form (BCNF) means in MySQL and how to decompose tables that violate it while preserving data integrity.

---

## What Is BCNF?

Boyce-Codd Normal Form (BCNF) is a stricter version of Third Normal Form (3NF). A table is in BCNF when, for every functional dependency `X -> Y`, X is a superkey (a set of columns that uniquely identifies a row).

In other words: **every determinant must be a candidate key**.

Most tables in 3NF are also in BCNF. Violations occur only when a table has multiple overlapping candidate keys.

## Example of a BCNF Violation

Consider a college scheduling system:

```sql
-- A student can be enrolled in a course taught by only one teacher.
-- A teacher teaches only one course (but may teach multiple students).
-- The same course can be taught by different teachers.
-- Candidate keys: (student, teacher) and (student, course)
CREATE TABLE enrollments (
    student VARCHAR(50) NOT NULL,
    course  VARCHAR(50) NOT NULL,
    teacher VARCHAR(50) NOT NULL,
    PRIMARY KEY (student, course)
);
```

The functional dependency `teacher -> course` exists (each teacher teaches exactly one course), but `teacher` alone is not a superkey of this table. This violates BCNF.

**The anomaly**: if teacher "Dr. Smith" changes the course they teach, every row for Dr. Smith must be updated.

## Applying BCNF: Decompose the Table

```sql
-- teacher_courses: captures teacher -> course dependency
CREATE TABLE teacher_courses (
    teacher VARCHAR(50) NOT NULL PRIMARY KEY,
    course  VARCHAR(50) NOT NULL
);

-- student_teachers: captures which teacher a student studies with
CREATE TABLE student_teachers (
    student VARCHAR(50) NOT NULL,
    teacher VARCHAR(50) NOT NULL,
    PRIMARY KEY (student, teacher),
    CONSTRAINT fk_st_teacher FOREIGN KEY (teacher) REFERENCES teacher_courses (teacher)
);
```

Now the `teacher -> course` dependency is captured in `teacher_courses`, and there is no redundancy.

## Querying the BCNF Schema

```sql
-- Which course is a student enrolled in?
SELECT st.student, tc.course, st.teacher
FROM student_teachers st
JOIN teacher_courses tc ON tc.teacher = st.teacher
WHERE st.student = 'Alice';
```

## BCNF vs 3NF Trade-offs

Decomposing to BCNF can sometimes lose the ability to enforce a dependency through a single table constraint. In the example above, the original `(student, course)` candidate key cannot be enforced as a simple unique constraint after decomposition. This is the classic trade-off between BCNF and 3NF.

```sql
-- To enforce "a student takes a course at most once" in the BCNF schema:
-- Use a VIEW or application logic, since the constraint spans both tables.
```

In practice, most MySQL schemas only encounter BCNF issues in complex many-to-many relationships with multiple overlapping keys. Always validate that decomposition does not lose critical constraints before committing to BCNF.

## Quick BCNF Checklist

```text
1. List all functional dependencies in the table.
2. Identify which dependencies have a determinant that is NOT a superkey.
3. For each violating dependency X -> Y, create a new table (X, Y) and remove Y from the original.
4. Verify that the decomposition is lossless (can be rejoined without spurious rows).
```

## Summary

BCNF strengthens 3NF by requiring that every functional dependency is determined by a superkey. Violations appear in tables with multiple candidate keys that overlap. Decompose violating tables by moving the non-superkey determinant and its dependents into a separate table. Be aware that BCNF decomposition can make some constraints harder to enforce and may require additional application-level validation.
