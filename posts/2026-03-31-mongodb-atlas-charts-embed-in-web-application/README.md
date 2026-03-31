# How to Embed Atlas Charts in Your Web Application

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Embedding, Web Application, SDK

Description: Learn how to embed MongoDB Atlas Charts into your web application using the Charts Embedding SDK with authentication and dynamic filter injection.

---

## Why Embed Atlas Charts

Instead of building charts from scratch with D3 or Chart.js, you can embed MongoDB Atlas Charts directly into your web app. The charts stay in sync with your live MongoDB data, support interactive filtering, and require no custom visualization code. This is ideal for product analytics dashboards, customer-facing reports, and internal admin panels.

## Embedding Methods

Atlas Charts supports two embedding approaches:

1. **Unauthenticated embedding**: charts are publicly accessible via a signed URL. No login required but the chart must have public access enabled.
2. **Authenticated embedding**: charts require a JWT token. Use this for private dashboards where access depends on the viewer's identity.

## Installing the Embedding SDK

```bash
npm install @mongodb-js/charts-embed-dom
```

## Unauthenticated Embed (Quick Start)

In Atlas Charts:
1. Open the chart, click the three-dot menu, and select **Embed Chart**
2. Enable **Unauthenticated Access**
3. Copy the **Chart ID** and **Base URL**

```javascript
import ChartsEmbedSDK from '@mongodb-js/charts-embed-dom';

const sdk = new ChartsEmbedSDK({
  baseUrl: 'https://charts.mongodb.com/charts-project-abc123'
});

const chart = sdk.createChart({
  chartId: 'your-chart-id-here',
  height: '400px'
});

chart.render(document.getElementById('chart-container'));
```

```html
<div id="chart-container"></div>
```

## Authenticated Embed with JWT

For authenticated embedding, your backend generates a signed JWT that Atlas Charts validates. The JWT can carry user identity claims that Atlas Charts uses to filter data via a custom user filter.

### Backend - Generating the JWT

```javascript
const jwt = require('jsonwebtoken');

// Route: GET /api/charts-token
app.get('/api/charts-token', authenticateUser, (req, res) => {
  const token = jwt.sign(
    {
      sub: req.user.id,
      email: req.user.email,
      orgId: req.user.organizationId  // used in Atlas Charts user filter
    },
    process.env.CHARTS_SIGNING_SECRET,
    { expiresIn: '1h', algorithm: 'HS256' }
  );
  res.json({ token });
});
```

### Frontend - Using the JWT

```javascript
const sdk = new ChartsEmbedSDK({
  baseUrl: 'https://charts.mongodb.com/charts-project-abc123',
  getUserToken: async () => {
    const response = await fetch('/api/charts-token');
    const { token } = await response.json();
    return token;
  }
});

const chart = sdk.createChart({
  chartId: 'your-chart-id-here',
  height: '400px'
});

await chart.render(document.getElementById('chart-container'));
```

The `getUserToken` function is called automatically when the chart loads and when the token expires.

## Injecting Dynamic Filters

You can programmatically filter the chart based on user selections in your app - without creating a full Atlas Charts dashboard:

```javascript
// Filter chart to show only the current user's data
await chart.setFilter({
  userId: currentUser.id,
  createdAt: {
    $gte: { $date: startDate.toISOString() },
    $lt:  { $date: endDate.toISOString() }
  }
});
```

The filter uses standard MongoDB query syntax and is merged server-side with any chart-level filters you set in the Atlas UI.

## Refreshing Charts

To trigger a data refresh programmatically (for example, after the user submits a form):

```javascript
await chart.refresh();
```

Or configure auto-refresh when creating the chart:

```javascript
const chart = sdk.createChart({
  chartId: 'your-chart-id-here',
  height: '400px',
  autoRefresh: true,
  maxDataAge: 60  // refresh if data is older than 60 seconds
});
```

## Handling Render Events

Listen for chart events to show loading states or handle errors:

```javascript
chart.addEventListener('loadingStateChanged', (event) => {
  document.getElementById('spinner').style.display =
    event.data.loading ? 'block' : 'none';
});

chart.addEventListener('dataError', (event) => {
  console.error('Chart data error:', event.data.error);
});
```

## React Component Wrapper

```javascript
import { useEffect, useRef } from 'react';
import ChartsEmbedSDK from '@mongodb-js/charts-embed-dom';

export function AtlasChart({ chartId, filter }) {
  const containerRef = useRef(null);
  const chartRef = useRef(null);

  useEffect(() => {
    const sdk = new ChartsEmbedSDK({
      baseUrl: process.env.NEXT_PUBLIC_CHARTS_BASE_URL
    });
    chartRef.current = sdk.createChart({ chartId, height: '400px' });
    chartRef.current.render(containerRef.current);
  }, [chartId]);

  useEffect(() => {
    if (chartRef.current && filter) {
      chartRef.current.setFilter(filter);
    }
  }, [filter]);

  return <div ref={containerRef} />;
}
```

## Summary

Embedding MongoDB Atlas Charts in a web application requires the `@mongodb-js/charts-embed-dom` SDK and either unauthenticated URL sharing or JWT-based authentication for private charts. Dynamic filter injection via `setFilter` lets your application pass user-specific or session-specific filter state to the chart without touching the Atlas UI. Auto-refresh and event listeners make embedded charts suitable for real-time operational dashboards in production web applications.
