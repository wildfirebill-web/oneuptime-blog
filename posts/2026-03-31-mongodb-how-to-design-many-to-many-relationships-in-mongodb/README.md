# How to Design Many-to-Many Relationships in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Data Modeling, Many-to-Many, Schema Design, Relationship

Description: Learn how to model many-to-many relationships in MongoDB using array references, junction collections, and embedded denormalization with practical examples.

---

## Many-to-Many in MongoDB

A many-to-many relationship exists when multiple documents on each side relate to multiple documents on the other side. Examples: students enrolled in many courses, products belonging to many categories, tags applied to many blog posts.

MongoDB does not have native join tables, so you implement many-to-many using arrays of references or junction collections.

## Approach 1: Array of References (Embedded IDs)

Store an array of referenced IDs in one or both sides of the relationship:

```javascript
// students collection
{
  _id: ObjectId("s001"),
  name: "Alice",
  courseIds: [ObjectId("c001"), ObjectId("c002"), ObjectId("c003")]
}

// courses collection
{
  _id: ObjectId("c001"),
  name: "MongoDB Fundamentals",
  studentIds: [ObjectId("s001"), ObjectId("s002")]
}
```

Find all courses for a student:

```javascript
const student = db.students.findOne({ _id: ObjectId("s001") });
db.courses.find({ _id: { $in: student.courseIds } }).toArray();
```

Index both sides for efficient lookups:

```javascript
db.courses.createIndex({ studentIds: 1 })
db.students.createIndex({ courseIds: 1 })
```

**When to use**: bounded relationships (small array sizes), data is accessed from both sides frequently, no extra metadata on the relationship.

## Approach 2: Junction Collection

Store the relationship in a dedicated junction collection, similar to a SQL join table:

```javascript
// enrollments collection (junction)
{
  _id: ObjectId("e001"),
  studentId: ObjectId("s001"),
  courseId: ObjectId("c001"),
  enrolledAt: ISODate("2026-01-10T00:00:00Z"),
  grade: "A",
  status: "active"
}
```

Index both foreign keys:

```javascript
db.enrollments.createIndex({ studentId: 1, courseId: 1 }, { unique: true })
db.enrollments.createIndex({ courseId: 1 })
```

Get all courses for a student with enrollment details:

```javascript
db.enrollments.aggregate([
  { $match: { studentId: ObjectId("s001") } },
  {
    $lookup: {
      from: "courses",
      localField: "courseId",
      foreignField: "_id",
      as: "course"
    }
  },
  { $unwind: "$course" }
])
```

**When to use**: relationship has metadata (grade, enrollment date, status), many-to-many is unbounded or grows large, you need to query the relationship itself.

## Approach 3: Denormalized Embedding (One-Sided)

For read-heavy applications with asymmetric access, denormalize tags or categories into each document:

```javascript
// articles collection with embedded tags
{
  _id: ObjectId("a001"),
  title: "MongoDB Best Practices",
  tags: ["mongodb", "nosql", "database"],
  categories: [
    { _id: ObjectId("cat001"), name: "Databases" },
    { _id: ObjectId("cat002"), name: "Backend" }
  ]
}
```

Create multikey index on tags for efficient filtering:

```javascript
db.articles.createIndex({ tags: 1 })
db.articles.createIndex({ "categories._id": 1 })
```

Query all articles with a specific tag:

```javascript
db.articles.find({ tags: "mongodb" })
```

**When to use**: tag-style relationships, read-heavy workloads, categories rarely change, query is always from the parent side.

## Choosing the Right Approach

```text
Factor                              | Array Refs  | Junction    | Embedded
------------------------------------|-------------|-------------|----------
Relationship has metadata           | No          | Yes         | No
Unbounded children                  | Risk        | Yes         | Yes (tags)
Query both sides independently      | Possible    | Yes         | No
Atomic updates needed               | No          | No          | Yes
Simple tag/category pattern         | No          | No          | Yes
```

## Updating Many-to-Many with Transactions

When using junction collections, ensure consistency with transactions:

```javascript
const session = db.getMongo().startSession();
session.startTransaction();
try {
  db.enrollments.insertOne({
    studentId: ObjectId("s001"),
    courseId: ObjectId("c001"),
    enrolledAt: new Date()
  }, { session });
  db.courses.updateOne(
    { _id: ObjectId("c001") },
    { $inc: { enrollmentCount: 1 } },
    { session }
  );
  session.commitTransaction();
} catch (e) {
  session.abortTransaction();
  throw e;
}
```

## Summary

Model many-to-many relationships in MongoDB using array references for simple bounded scenarios, junction collections when the relationship itself has metadata or is unbounded, and embedded denormalization for read-heavy tag-style patterns. Always index foreign key arrays and junction collection fields for efficient lookups. Choose based on whether relationships need metadata, how large they can grow, and from which side queries are most common.
