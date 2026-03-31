# How to Build a Live Dashboard with MongoDB Change Streams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Change Stream, Dashboard, Real-Time, Node

Description: Learn how to build a live analytics dashboard that updates in real time by tailing MongoDB change streams and pushing aggregated metrics to clients via SSE.

---

Live dashboards display metrics that update as the underlying data changes. MongoDB change streams let your server react to every insert or update and push refreshed aggregations to browser clients. This guide uses Server-Sent Events (SSE) for a lightweight one-way push channel.

## Architecture Overview

```text
Browser <-- SSE -- Node.js Server <-- Change Stream -- MongoDB
```

SSE is simpler than WebSockets for unidirectional data push and works through standard HTTP.

## Installing Dependencies

```bash
npm install express mongoose
```

## Defining Mongoose Models

```javascript
const mongoose = require('mongoose');

const saleSchema = new mongoose.Schema({
  productId: mongoose.Schema.Types.ObjectId,
  category: String,
  amount: Number,
  region: String,
  createdAt: { type: Date, default: Date.now },
});

const Sale = mongoose.model('Sale', saleSchema);
```

## Computing Dashboard Metrics

```javascript
async function getDashboardMetrics() {
  const [revenueResult, salesByCategory, salesByRegion, recentSales] = await Promise.all([
    Sale.aggregate([
      { $group: { _id: null, total: { $sum: '$amount' }, count: { $sum: 1 } } },
    ]),
    Sale.aggregate([
      { $group: { _id: '$category', revenue: { $sum: '$amount' }, count: { $sum: 1 } } },
      { $sort: { revenue: -1 } },
    ]),
    Sale.aggregate([
      { $group: { _id: '$region', revenue: { $sum: '$amount' } } },
      { $sort: { revenue: -1 } },
    ]),
    Sale.find().sort({ createdAt: -1 }).limit(10).lean(),
  ]);

  return {
    totalRevenue: revenueResult[0]?.total ?? 0,
    totalSales: revenueResult[0]?.count ?? 0,
    byCategory: salesByCategory,
    byRegion: salesByRegion,
    recentSales,
    updatedAt: new Date(),
  };
}
```

## SSE Endpoint

```javascript
const express = require('express');
const app = express();

// Keep track of connected SSE clients
const clients = new Set();

app.get('/dashboard/stream', async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('Access-Control-Allow-Origin', '*');

  // Send initial metrics
  const metrics = await getDashboardMetrics();
  res.write(`data: ${JSON.stringify(metrics)}\n\n`);

  clients.add(res);

  req.on('close', () => {
    clients.delete(res);
  });
});

function broadcastMetrics(metrics) {
  const payload = `data: ${JSON.stringify(metrics)}\n\n`;
  for (const client of clients) {
    client.write(payload);
  }
}
```

## Tailing Change Streams to Trigger Updates

```javascript
function watchSales() {
  const stream = Sale.watch([], { fullDocument: 'updateLookup' });

  let debounceTimer = null;

  stream.on('change', () => {
    // Debounce to avoid recalculating on every single insert during bulk loads
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(async () => {
      const metrics = await getDashboardMetrics();
      broadcastMetrics(metrics);
    }, 300);
  });

  stream.on('error', (err) => {
    console.error('Change stream error:', err.message);
    setTimeout(watchSales, 5000);
  });
}

mongoose.connection.once('open', () => {
  watchSales();
  app.listen(3000, () => console.log('Dashboard server on port 3000'));
});
```

## Browser Client

```javascript
const source = new EventSource('/dashboard/stream');

source.onmessage = (event) => {
  const metrics = JSON.parse(event.data);
  document.getElementById('total-revenue').textContent =
    `$${metrics.totalRevenue.toLocaleString()}`;
  renderCategoryChart(metrics.byCategory);
  renderRegionMap(metrics.byRegion);
};

source.onerror = () => {
  console.warn('SSE connection lost, will retry...');
};
```

## Filtering the Change Stream by Collection

To watch multiple collections and broadcast different metrics:

```javascript
const mongoose = require('mongoose');

const salesStream = Sale.watch();
const ordersStream = Order.watch();

salesStream.on('change', () => broadcastSalesMetrics());
ordersStream.on('change', () => broadcastOrderMetrics());
```

## Summary

MongoDB change streams paired with SSE provide a lightweight pattern for live dashboards. Debounce the change stream handler to batch rapid changes into a single aggregation query, and use SSE instead of WebSockets when the data flow is strictly server-to-client. Reconnect the change stream automatically after errors to handle replica set elections without downtime.
