# How to Implement Event Sequencing with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Event Sequencing, Ordering, Pub/Sub, State Management

Description: Implement event sequencing with Dapr to ensure out-of-order events are reordered and processed in the correct sequence using state-based buffering and sequence tracking.

---

## Overview

Distributed messaging systems do not guarantee event ordering. Dapr pub/sub may deliver events out of order, especially during retries. Event sequencing buffers out-of-order events and processes them in the correct sequence number order.

## Sequence Number Tracking

Track and enforce event sequence numbers per entity:

```javascript
const { DaprServer, DaprClient } = require('@dapr/dapr');
const server = new DaprServer();
const client = new DaprClient();

const SEQUENCE_BUFFER_TTL = 300; // 5 minutes

await server.pubsub.subscribe('pubsub', 'account-events', async (event) => {
  const { accountId, sequenceNum, payload } = event;
  await processSequencedEvent(accountId, sequenceNum, payload);
});

async function processSequencedEvent(entityId, seqNum, payload) {
  const seqKey = `seq-state-${entityId}`;
  const bufferKey = `seq-buffer-${entityId}`;

  const rawSeq = await client.state.get('statestore', seqKey);
  const seqState = rawSeq ? JSON.parse(rawSeq) : { nextExpected: 1 };

  if (seqNum === seqState.nextExpected) {
    // In-order - process immediately
    await handleEvent(entityId, seqNum, payload);
    seqState.nextExpected += 1;

    // Check buffer for subsequent events
    let nextFromBuffer = true;
    while (nextFromBuffer) {
      const rawBuffer = await client.state.get('statestore', bufferKey);
      const buffer = rawBuffer ? JSON.parse(rawBuffer) : [];
      const nextIdx = buffer.findIndex(e => e.seqNum === seqState.nextExpected);

      if (nextIdx !== -1) {
        const nextEvent = buffer.splice(nextIdx, 1)[0];
        await handleEvent(entityId, nextEvent.seqNum, nextEvent.payload);
        seqState.nextExpected += 1;
        await client.state.save('statestore', [{ key: bufferKey, value: JSON.stringify(buffer), metadata: { ttlInSeconds: String(SEQUENCE_BUFFER_TTL) } }]);
      } else {
        nextFromBuffer = false;
      }
    }

    await client.state.save('statestore', [{ key: seqKey, value: JSON.stringify(seqState) }]);
  } else if (seqNum > seqState.nextExpected) {
    // Out-of-order - buffer it
    const rawBuffer = await client.state.get('statestore', bufferKey);
    const buffer = rawBuffer ? JSON.parse(rawBuffer) : [];

    if (!buffer.find(e => e.seqNum === seqNum)) {
      buffer.push({ seqNum, payload });
    }

    await client.state.save('statestore', [{ key: bufferKey, value: JSON.stringify(buffer), metadata: { ttlInSeconds: String(SEQUENCE_BUFFER_TTL) } }]);
    console.log(`Buffered out-of-order event ${seqNum} for ${entityId}, waiting for ${seqState.nextExpected}`);
  } else {
    // Duplicate or old event - discard
    console.log(`Discarding old event ${seqNum} for ${entityId}, already processed up to ${seqState.nextExpected - 1}`);
  }
}

async function handleEvent(entityId, seqNum, payload) {
  console.log(`Processing event ${seqNum} for ${entityId}`);
  // Your business logic here
}
```

## Gap Detection and Timeout

Handle sequences with permanent gaps (missing events):

```javascript
async function checkForSequenceGaps(entityId) {
  const bufferKey = `seq-buffer-${entityId}`;
  const seqKey = `seq-state-${entityId}`;

  const rawSeq = await client.state.get('statestore', seqKey);
  const rawBuffer = await client.state.get('statestore', bufferKey);

  if (!rawSeq || !rawBuffer) return;

  const seqState = JSON.parse(rawSeq);
  const buffer = JSON.parse(rawBuffer);

  const oldestBuffered = buffer.reduce((min, e) => Math.min(min, e.seqNum), Infinity);
  const gapSize = oldestBuffered - seqState.nextExpected;

  if (gapSize > 10) {
    console.warn(`Sequence gap of ${gapSize} detected for ${entityId}. Skipping to ${oldestBuffered}`);
    seqState.nextExpected = oldestBuffered;
    await client.state.save('statestore', [{ key: seqKey, value: JSON.stringify(seqState) }]);
  }
}
```

## Summary

Event sequencing with Dapr combines pub/sub for event ingestion with state management for buffering out-of-order events and tracking expected sequence numbers. Out-of-order events are held in a buffer until their predecessors arrive, then processed in order. Set buffer TTLs and implement gap detection to prevent buffers from growing indefinitely when events are permanently lost.
