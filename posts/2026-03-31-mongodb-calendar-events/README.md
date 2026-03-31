# How to Implement Calendar Events with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Calendar, Event, Date, Index

Description: Learn how to design a MongoDB schema for calendar events, handle attendees, query by date range, and build efficient calendar views with aggregation.

---

## Event Schema Design

A calendar event schema must support single events, recurring series, attendees, and RSVP state. Here is a practical schema:

```javascript
{
  _id: ObjectId("..."),
  title: "Q2 Planning Session",
  description: "Quarterly roadmap review",
  organizerId: "user-001",
  attendees: [
    { userId: "user-002", status: "accepted" },
    { userId: "user-003", status: "pending" },
    { userId: "user-004", status: "declined" }
  ],
  start:    ISODate("2026-04-15T14:00:00Z"),
  end:      ISODate("2026-04-15T16:00:00Z"),
  timezone: "UTC",
  location: "Conference Room B",
  isAllDay: false,
  visibility: "private",   // "public" | "private"
  tags: ["planning", "q2"],
  createdAt: ISODate("2026-03-31T10:00:00Z"),
  updatedAt: ISODate("2026-03-31T10:00:00Z")
}
```

## Indexes for Calendar Queries

```javascript
// Query by organizer + date range
db.events.createIndex({ organizerId: 1, start: 1 });

// Query events an attendee is invited to
db.events.createIndex({ "attendees.userId": 1, start: 1 });

// Full-text search on title/description
db.events.createIndex({ title: "text", description: "text" });
```

## Fetching Events for a Month View

```javascript
async function getMonthEvents(userId, year, month) {
  const start = new Date(Date.UTC(year, month - 1, 1));
  const end   = new Date(Date.UTC(year, month, 1));

  return db.collection("events").find({
    $or: [
      { organizerId: userId },
      { "attendees.userId": userId }
    ],
    start: { $lt: end },
    end:   { $gte: start }   // includes events spanning month boundary
  }).sort({ start: 1 }).toArray();
}
```

The overlap condition (`start < rangeEnd AND end >= rangeStart`) correctly captures events that span the range boundary.

## Building a Weekly Agenda View with Aggregation

```javascript
const weekStart = new Date("2026-04-13T00:00:00Z");
const weekEnd   = new Date("2026-04-20T00:00:00Z");

db.collection("events").aggregate([
  {
    $match: {
      "attendees.userId": "user-002",
      start: { $gte: weekStart, $lt: weekEnd }
    }
  },
  {
    $project: {
      title: 1,
      start: 1,
      end: 1,
      dayOfWeek: { $dayOfWeek: "$start" },
      durationMinutes: {
        $divide: [{ $subtract: ["$end", "$start"] }, 60000]
      },
      myStatus: {
        $let: {
          vars: { me: { $filter: { input: "$attendees", as: "a", cond: { $eq: ["$$a.userId", "user-002"] } } } },
          in: { $arrayElemAt: ["$$me.status", 0] }
        }
      }
    }
  },
  { $sort: { start: 1 } }
])
```

## Updating RSVP Status

```javascript
await db.collection("events").updateOne(
  { _id: eventId, "attendees.userId": userId },
  {
    $set: {
      "attendees.$.status": "accepted",
      updatedAt: new Date()
    }
  }
);
```

Using the positional `$` operator updates only the matching attendee subdocument.

## Searching Events by Title

```javascript
// Text search across title and description
const results = await db.collection("events").find(
  { $text: { $search: "planning roadmap" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } }).limit(20).toArray();
```

## Handling All-Day Events

All-day events span a full calendar day in local time. Store them with a flag and use date-only strings for reliable querying:

```javascript
{
  title: "Company Holiday",
  isAllDay: true,
  dateStr: "2026-07-04",   // store as string for all-day events
  start: ISODate("2026-07-04T00:00:00Z"),
  end:   ISODate("2026-07-05T00:00:00Z")
}
```

Query all-day events for a range using `dateStr` directly:

```javascript
db.events.find({ isAllDay: true, dateStr: { $gte: "2026-07-01", $lte: "2026-07-31" } })
```

## Summary

Implementing calendar events in MongoDB involves a flexible schema with attendees as embedded subdocuments, compound indexes on user ID and start date, and overlap-aware range queries (`start < rangeEnd AND end >= rangeStart`). Aggregation pipelines enable rich agenda views with computed fields like duration and RSVP status. All-day events benefit from a supplementary date string field for simple string-range queries.
