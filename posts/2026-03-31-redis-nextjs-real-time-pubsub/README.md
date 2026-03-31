# How to Build a Next.js Real-Time App with Redis Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Next.js, Pub/Sub, Real-Time, Server-Sent Event

Description: Build a real-time Next.js app using Redis Pub/Sub and Server-Sent Events to push live updates from API routes to browser clients.

---

Next.js API routes are serverless-style functions, but they can hold long-lived connections for Server-Sent Events. Combining this with Redis Pub/Sub creates a real-time push channel that works across multiple Next.js instances.

## Install Dependencies

```bash
npm install redis
```

## Publisher: Post Notifications via API Route

`pages/api/notify.js`:

```javascript
import { createClient } from "redis";

const publisher = createClient({ url: process.env.REDIS_URL });
publisher.connect();

export default async function handler(req, res) {
  if (req.method !== "POST") return res.status(405).end();

  const { channel, message } = req.body;
  await publisher.publish(channel, JSON.stringify(message));
  res.json({ published: true });
}
```

## Subscriber: Server-Sent Events Stream

`pages/api/stream/[channel].js`:

```javascript
import { createClient } from "redis";

export const config = { api: { bodyParser: false } };

export default async function handler(req, res) {
  const { channel } = req.query;

  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");
  res.setHeader("X-Accel-Buffering", "no");
  res.flushHeaders();

  const subscriber = createClient({ url: process.env.REDIS_URL });
  await subscriber.connect();

  await subscriber.subscribe(channel, (message) => {
    res.write(`data: ${message}\n\n`);
  });

  // Clean up on client disconnect
  req.on("close", async () => {
    await subscriber.unsubscribe(channel);
    await subscriber.quit();
    res.end();
  });
}
```

## React Component: Listen for Updates

```javascript
// components/LiveFeed.jsx
"use client";
import { useEffect, useState } from "react";

export default function LiveFeed({ channel }) {
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    const eventSource = new EventSource(`/api/stream/${channel}`);

    eventSource.onmessage = (e) => {
      const data = JSON.parse(e.data);
      setMessages((prev) => [data, ...prev].slice(0, 50));
    };

    return () => eventSource.close();
  }, [channel]);

  return (
    <ul>
      {messages.map((msg, i) => (
        <li key={i}>{JSON.stringify(msg)}</li>
      ))}
    </ul>
  );
}
```

## Send a Notification

```bash
curl -X POST http://localhost:3000/api/notify \
  -H "Content-Type: application/json" \
  -d '{"channel":"notifications","message":{"text":"New order received","orderId":"42"}}'
```

## Scale Across Instances

Each Next.js instance has its own SSE connections. Redis Pub/Sub fans out messages to all subscriber connections across all instances, so every connected browser receives the notification regardless of which instance it connected to.

## Summary

Redis Pub/Sub combined with Next.js Server-Sent Events creates a scalable real-time push system without WebSocket complexity. Publishers call a simple API route, and each subscriber connection creates a dedicated Redis subscription. The approach scales horizontally because Redis broadcasts to all subscriber instances, which then forward messages to their connected SSE clients.
