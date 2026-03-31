# How to Build a Task Management Application with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema, Index, Aggregation, Application

Description: Learn how to model tasks, assignees, and statuses in MongoDB and build efficient queries for a task management application with filtering and reporting.

---

Task management applications require flexible schemas that can accommodate varying task types, multi-user assignment, priority levels, and real-time status updates. MongoDB's document model maps naturally to these requirements.

## Designing the Task Schema

A task document should include metadata for filtering, sorting, and reporting in a single collection.

```javascript
db.tasks.insertOne({
  title: "Design login screen",
  description: "Create wireframes and finalize the UI for the login page.",
  status: "in_progress",
  priority: "high",
  assignees: ["user-101", "user-204"],
  tags: ["frontend", "design"],
  dueDate: new Date("2026-04-15"),
  createdBy: "user-001",
  createdAt: new Date(),
  updatedAt: new Date(),
  subtasks: [
    { id: "st-1", title: "Wireframe", done: false },
    { id: "st-2", title: "Color palette", done: true }
  ]
});
```

## Creating Indexes for Common Queries

Tasks are typically queried by status, assignee, due date, or a combination. Create indexes that support these access patterns.

```javascript
db.tasks.createIndex({ status: 1, dueDate: 1 });
db.tasks.createIndex({ assignees: 1, status: 1 });
db.tasks.createIndex({ createdBy: 1, createdAt: -1 });
```

The `assignees` index supports multi-valued array fields out of the box because MongoDB creates a multikey index automatically.

## Querying Tasks for a User

Retrieve all open tasks assigned to a specific user, sorted by due date.

```javascript
db.tasks.find(
  {
    assignees: "user-101",
    status: { $in: ["todo", "in_progress"] }
  },
  { title: 1, status: 1, dueDate: 1, priority: 1 }
).sort({ dueDate: 1 });
```

## Updating Task Status

Use `$set` with `updatedAt` to track when a task was last modified.

```javascript
db.tasks.updateOne(
  { _id: ObjectId("...") },
  {
    $set: {
      status: "done",
      updatedAt: new Date()
    }
  }
);
```

## Marking a Subtask Complete

Use the positional `$` operator to update a specific subtask within the embedded array.

```javascript
db.tasks.updateOne(
  { _id: ObjectId("..."), "subtasks.id": "st-1" },
  { $set: { "subtasks.$.done": true, updatedAt: new Date() } }
);
```

## Reporting Overdue Tasks by Priority

Use the aggregation pipeline to count overdue tasks grouped by priority level.

```javascript
db.tasks.aggregate([
  {
    $match: {
      status: { $ne: "done" },
      dueDate: { $lt: new Date() }
    }
  },
  {
    $group: {
      _id: "$priority",
      count: { $sum: 1 },
      tasks: { $push: "$title" }
    }
  },
  { $sort: { count: -1 } }
]);
```

## Summary

A task management application in MongoDB works well with a single tasks collection that embeds subtasks and stores assignees as an array. Index on `assignees`, `status`, and `dueDate` to keep queries fast as the collection grows. Use `$set` with an `updatedAt` field to maintain a reliable audit trail of state changes, and use aggregation with `$group` to generate management reports without a separate analytics database.
