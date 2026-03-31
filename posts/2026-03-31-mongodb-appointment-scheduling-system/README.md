# How to Build an Appointment Scheduling System with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Scheduling, Date, Index, Transaction

Description: Learn how to build an appointment scheduling system in MongoDB with time slot management, conflict detection, and atomic booking using transactions and indexes.

---

## Data Model Design

An appointment scheduling system needs to track availability slots and bookings. A good schema separates provider availability (recurring patterns) from individual booked appointments:

```javascript
// Provider availability slot
{
  _id: ObjectId("..."),
  providerId: "dr-smith",
  start: ISODate("2026-04-01T09:00:00Z"),
  end:   ISODate("2026-04-01T09:30:00Z"),
  status: "available",   // "available" | "booked" | "blocked"
  __v: 0                 // optimistic locking version
}

// Appointment document
{
  _id: ObjectId("..."),
  slotId:     ObjectId("..."),
  providerId: "dr-smith",
  patientId:  "patient-456",
  start:      ISODate("2026-04-01T09:00:00Z"),
  end:        ISODate("2026-04-01T09:30:00Z"),
  status:     "confirmed",
  createdAt:  ISODate("2026-03-31T10:00:00Z")
}
```

## Creating Indexes for Efficient Slot Queries

```javascript
// Find available slots for a provider in a time range
db.slots.createIndex({ providerId: 1, start: 1, status: 1 });

// Detect overlapping bookings
db.appointments.createIndex({ providerId: 1, start: 1, end: 1 });
```

## Querying Available Slots

```javascript
async function getAvailableSlots(providerId, fromDate, toDate) {
  return db.collection("slots").find({
    providerId,
    status: "available",
    start: { $gte: fromDate, $lt: toDate }
  }).sort({ start: 1 }).toArray();
}
```

## Atomic Slot Booking

Booking a slot requires atomically checking availability and marking it taken to prevent double-booking:

```javascript
async function bookSlot(slotId, patientId) {
  const session = client.startSession();
  try {
    let appointmentId;
    await session.withTransaction(async () => {
      // Atomically claim the slot
      const slot = await db.collection("slots").findOneAndUpdate(
        { _id: slotId, status: "available" },
        { $set: { status: "booked" }, $inc: { __v: 1 } },
        { returnDocument: "after", session }
      );

      if (!slot) {
        throw new Error("Slot is no longer available");
      }

      // Create the appointment
      const result = await db.collection("appointments").insertOne(
        {
          slotId: slot._id,
          providerId: slot.providerId,
          patientId,
          start: slot.start,
          end: slot.end,
          status: "confirmed",
          createdAt: new Date()
        },
        { session }
      );
      appointmentId = result.insertedId;
    });
    return appointmentId;
  } finally {
    await session.endSession();
  }
}
```

## Generating Slots for a Provider

For providers with regular schedules, generate slots programmatically:

```javascript
async function generateSlots(providerId, date, slotDurationMinutes = 30, startHour = 9, endHour = 17) {
  const slots = [];
  const d = new Date(date);
  d.setUTCHours(startHour, 0, 0, 0);

  while (d.getUTCHours() < endHour) {
    const start = new Date(d);
    d.setMinutes(d.getMinutes() + slotDurationMinutes);
    const end = new Date(d);
    slots.push({ providerId, start, end, status: "available", __v: 0 });
  }

  await db.collection("slots").insertMany(slots);
}
```

## Cancelling and Releasing Slots

```javascript
async function cancelAppointment(appointmentId) {
  const session = client.startSession();
  try {
    await session.withTransaction(async () => {
      const appt = await db.collection("appointments").findOneAndUpdate(
        { _id: appointmentId, status: "confirmed" },
        { $set: { status: "cancelled" } },
        { session }
      );
      if (!appt) throw new Error("Appointment not found or already cancelled");

      await db.collection("slots").updateOne(
        { _id: appt.slotId },
        { $set: { status: "available" } },
        { session }
      );
    });
  } finally {
    await session.endSession();
  }
}
```

## Summary

A MongoDB appointment scheduling system uses a slots collection (with status tracking) and an appointments collection linked to booked slots. Compound indexes on `providerId + start + status` enable fast availability queries. Atomic booking is implemented with `findOneAndUpdate` inside a transaction to prevent double-booking. Slot generation and cancellation complete the lifecycle, ensuring slot status always stays consistent with appointment records.
