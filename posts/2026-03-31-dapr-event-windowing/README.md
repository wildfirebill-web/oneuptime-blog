# How to Implement Event Windowing with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Event Windowing, Stream Processing, Time Window, Analytics

Description: Implement tumbling, sliding, and session windows for Dapr pub/sub event streams using state management to compute time-based aggregations and analytics.

---

## Overview

Event windowing groups streaming events into bounded time periods for aggregation. Dapr supports windowing patterns through its state management API - you track window boundaries in state and flush results when windows close.

## Tumbling Windows

Tumbling windows divide time into fixed, non-overlapping intervals:

```javascript
const { DaprServer, DaprClient } = require('@dapr/dapr');
const client = new DaprClient();

const WINDOW_SIZE_MS = 60 * 1000; // 1-minute tumbling window

async function processEventInTumblingWindow(event) {
  const windowId = Math.floor(event.timestamp / WINDOW_SIZE_MS);
  const windowKey = `tumbling-window-${windowId}`;

  const raw = await client.state.get('statestore', windowKey);
  const window = raw ? JSON.parse(raw) : {
    windowId,
    start: windowId * WINDOW_SIZE_MS,
    end: (windowId + 1) * WINDOW_SIZE_MS,
    events: [],
    count: 0,
    sum: 0
  };

  window.events.push(event);
  window.count += 1;
  window.sum += event.value;
  window.avg = window.sum / window.count;

  await client.state.save('statestore', [{
    key: windowKey,
    value: JSON.stringify(window),
    metadata: { ttlInSeconds: '180' }
  }]);

  return window;
}
```

## Sliding Windows

Sliding windows advance by a step smaller than the window size:

```javascript
const WINDOW_DURATION_MS = 5 * 60 * 1000;  // 5 minutes
const SLIDE_INTERVAL_MS = 60 * 1000;         // 1 minute slide

async function processEventInSlidingWindow(event) {
  const now = event.timestamp;
  const windowStart = now - (now % SLIDE_INTERVAL_MS);

  // Event belongs to multiple overlapping windows
  const windowIds = [];
  for (let start = windowStart - WINDOW_DURATION_MS + SLIDE_INTERVAL_MS;
       start <= windowStart;
       start += SLIDE_INTERVAL_MS) {
    windowIds.push(start);
  }

  for (const wStart of windowIds) {
    const windowKey = `sliding-window-${wStart}`;
    const raw = await client.state.get('statestore', windowKey);
    const window = raw ? JSON.parse(raw) : { start: wStart, end: wStart + WINDOW_DURATION_MS, events: [] };
    window.events.push(event);
    await client.state.save('statestore', [{ key: windowKey, value: JSON.stringify(window), metadata: { ttlInSeconds: '600' } }]);
  }
}
```

## Session Windows

Session windows group events separated by less than a gap duration:

```javascript
const SESSION_GAP_MS = 30 * 60 * 1000; // 30-minute inactivity gap

async function processEventInSessionWindow(userId, event) {
  const sessionKey = `session-${userId}`;
  const raw = await client.state.get('statestore', sessionKey);
  const session = raw ? JSON.parse(raw) : {
    userId,
    sessionId: `${userId}-${Date.now()}`,
    events: [],
    startTime: event.timestamp,
    lastEventTime: 0
  };

  const timeSinceLast = event.timestamp - session.lastEventTime;
  if (timeSinceLast > SESSION_GAP_MS && session.events.length > 0) {
    // Close old session
    await client.pubsub.publish('pubsub', 'session-closed', session);
    // Start new session
    session.sessionId = `${userId}-${Date.now()}`;
    session.events = [];
    session.startTime = event.timestamp;
  }

  session.events.push(event);
  session.lastEventTime = event.timestamp;

  await client.state.save('statestore', [{ key: sessionKey, value: JSON.stringify(session), metadata: { ttlInSeconds: '3600' } }]);
}
```

## Flushing Windows on Schedule

Use a Dapr cron binding to flush expired windows:

```javascript
app.post('/flush-windows', async (req, res) => {
  const expiredWindowId = Math.floor(Date.now() / WINDOW_SIZE_MS) - 2;
  const expiredKey = `tumbling-window-${expiredWindowId}`;

  const raw = await client.state.get('statestore', expiredKey);
  if (raw) {
    const window = JSON.parse(raw);
    await client.pubsub.publish('pubsub', 'window-results', window);
    await client.state.delete('statestore', expiredKey);
  }

  res.status(200).send('OK');
});
```

## Summary

Dapr enables time-based event windowing using state management to track window state and cron bindings to trigger periodic flushes. Tumbling windows are easiest to implement - events map to exactly one window. Sliding windows require events to be added to multiple overlapping windows. Session windows track per-user activity gaps and are ideal for analyzing user behavior patterns.
