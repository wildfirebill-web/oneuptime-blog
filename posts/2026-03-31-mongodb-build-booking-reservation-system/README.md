# How to Build a Booking and Reservation System with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Booking, Reservation, Transaction, Schema Design

Description: Learn how to build a booking and reservation system with MongoDB using atomic transactions to prevent double-booking and manage availability windows.

---

## Schema Design

A booking system must prevent double-bookings while remaining performant under concurrent load. The core approach is to model resources with an availability array and use transactions for atomic reservation.

```javascript
// Resource document (e.g., hotel room, meeting room, vehicle)
{
  _id: ObjectId(),
  name: "Conference Room A",
  type: "meeting_room",
  capacity: 10,
  amenities: ["projector", "whiteboard"],
  pricePerHour: 50,
  isActive: true,
  bookedSlots: [
    {
      bookingId: ObjectId("..."),
      start: ISODate("2026-04-01T09:00:00Z"),
      end: ISODate("2026-04-01T11:00:00Z"),
    }
  ]
}

// Booking document
{
  _id: ObjectId(),
  resourceId: ObjectId("..."),
  userId: ObjectId("..."),
  status: "confirmed",    // "pending", "confirmed", "cancelled"
  start: ISODate("2026-04-01T09:00:00Z"),
  end: ISODate("2026-04-01T11:00:00Z"),
  totalPrice: 100,
  notes: "Team standup",
  createdAt: ISODate(),
  updatedAt: ISODate()
}
```

## Indexes

```javascript
async function setupBookingSystem(db) {
  // Index for availability queries
  await db.collection('resources').createIndex({ type: 1, isActive: 1 });
  await db.collection('resources').createIndex({ 'bookedSlots.start': 1, 'bookedSlots.end': 1 });

  // Booking lookups
  await db.collection('bookings').createIndex({ userId: 1, start: -1 });
  await db.collection('bookings').createIndex({ resourceId: 1, start: 1, end: 1 });
  await db.collection('bookings').createIndex({ status: 1, start: 1 });
}
```

## Checking Availability

```javascript
async function checkAvailability(db, { resourceId, start, end }) {
  const resource = await db.collection('resources').findOne({
    _id: resourceId,
    isActive: true,
    // Check that no existing slot overlaps the requested range
    bookedSlots: {
      $not: {
        $elemMatch: {
          start: { $lt: end },
          end: { $gt: start },
        },
      },
    },
  });
  return resource !== null;
}
```

## Atomic Booking with Transaction

```javascript
async function createBooking(db, client, { resourceId, userId, start, end, notes }) {
  const session = client.startSession();
  let bookingId;

  await session.withTransaction(async () => {
    const startDate = new Date(start);
    const endDate = new Date(end);
    const hours = (endDate - startDate) / 3600000;

    // Fetch resource and verify availability inside the transaction
    const resource = await db.collection('resources').findOne(
      {
        _id: resourceId,
        isActive: true,
        bookedSlots: {
          $not: { $elemMatch: { start: { $lt: endDate }, end: { $gt: startDate } } },
        },
      },
      { session }
    );

    if (!resource) throw new Error('Resource not available for the requested time');

    const totalPrice = resource.pricePerHour * hours;
    const bookingDoc = {
      resourceId,
      userId,
      status: 'confirmed',
      start: startDate,
      end: endDate,
      totalPrice,
      notes,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    // Insert booking
    const bookingResult = await db.collection('bookings').insertOne(bookingDoc, { session });
    bookingId = bookingResult.insertedId;

    // Add slot to resource's bookedSlots array
    await db.collection('resources').updateOne(
      { _id: resourceId },
      { $push: { bookedSlots: { bookingId, start: startDate, end: endDate } } },
      { session }
    );
  });

  await session.endSession();
  return bookingId;
}
```

## Cancelling a Booking

```javascript
async function cancelBooking(db, client, bookingId, userId) {
  const session = client.startSession();

  await session.withTransaction(async () => {
    const booking = await db.collection('bookings').findOne(
      { _id: bookingId, userId, status: 'confirmed' },
      { session }
    );
    if (!booking) throw new Error('Booking not found or already cancelled');

    await db.collection('bookings').updateOne(
      { _id: bookingId },
      { $set: { status: 'cancelled', updatedAt: new Date() } },
      { session }
    );

    await db.collection('resources').updateOne(
      { _id: booking.resourceId },
      { $pull: { bookedSlots: { bookingId } } },
      { session }
    );
  });

  await session.endSession();
}
```

## Summary

Use MongoDB transactions to prevent double-bookings by combining availability check and slot reservation in a single atomic operation. Store booked slots as an embedded array in the resource document for fast overlap queries using `$elemMatch`. Use `$pull` for cancellations to remove slots atomically, keeping the resource document consistent with the bookings collection.
