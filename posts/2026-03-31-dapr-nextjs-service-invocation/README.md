# How to Use Dapr with Next.js and Dapr Service Invocation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Next.js, Service Invocation, Node.js, Full Stack

Description: Integrate Dapr service invocation with Next.js API routes to build server-side microservice communication with built-in resilience and tracing.

---

Next.js API routes are ideal for server-side Dapr service invocation because calls happen server-side where the Dapr sidecar runs, avoiding browser CORS issues. This guide shows how to use Next.js API routes as a BFF (Backend for Frontend) that calls downstream microservices via Dapr.

## Architecture

```text
Browser -> Next.js Page -> Next.js API Route (with Dapr sidecar) -> Backend Services
```

The Next.js app runs with a co-located Dapr sidecar. API routes call other services via Dapr HTTP, while pages use standard Next.js data fetching.

## Setting Up Next.js with Dapr

Create a Next.js project and install the Dapr client:

```bash
npx create-next-app@latest shop-frontend --typescript
cd shop-frontend
npm install @dapr/dapr
```

## Creating Dapr-Enabled API Routes

Create an API route that calls a product service via Dapr:

```typescript
// pages/api/products/index.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { DaprClient, HttpMethod } from '@dapr/dapr';

const daprHost = process.env.DAPR_HTTP_HOST || 'localhost';
const daprPort = process.env.DAPR_HTTP_PORT || '3500';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const client = new DaprClient({ daprHost, daprPort });

  if (req.method === 'GET') {
    try {
      const products = await client.invoker.invoke(
        'product-service',
        'products',
        HttpMethod.GET
      );
      res.status(200).json(products);
    } catch (error) {
      res.status(502).json({ error: 'Failed to fetch products' });
    }
  } else if (req.method === 'POST') {
    try {
      const product = await client.invoker.invoke(
        'product-service',
        'products',
        HttpMethod.POST,
        req.body
      );
      res.status(201).json(product);
    } catch (error) {
      res.status(502).json({ error: 'Failed to create product' });
    }
  }
}
```

Create an orders API route that invokes multiple services:

```typescript
// pages/api/orders/index.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { DaprClient, HttpMethod } from '@dapr/dapr';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const client = new DaprClient({
    daprHost: process.env.DAPR_HTTP_HOST || 'localhost',
    daprPort: process.env.DAPR_HTTP_PORT || '3500',
  });

  if (req.method === 'POST') {
    const order = req.body;

    // Validate inventory via Dapr service invocation
    const inventory = await client.invoker.invoke(
      'inventory-service',
      `inventory/${order.productId}`,
      HttpMethod.GET
    );

    if (inventory.quantity < order.quantity) {
      return res.status(409).json({ error: 'Insufficient inventory' });
    }

    // Create the order
    const created = await client.invoker.invoke(
      'order-service',
      'orders',
      HttpMethod.POST,
      order
    );

    // Publish order created event
    await client.pubsub.publish('pubsub', 'order-created', created);

    res.status(201).json(created);
  }
}
```

## Using Dapr State in Next.js API Routes

```typescript
// pages/api/cart/[userId].ts
import { NextApiRequest, NextApiResponse } from 'next';
import { DaprClient } from '@dapr/dapr';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { userId } = req.query;
  const client = new DaprClient();

  const cart = await client.state.get('statestore', `cart-${userId}`);
  res.status(200).json(cart || { items: [] });
}
```

## Next.js Page with Server-Side Dapr Fetching

```typescript
// pages/products.tsx
import { GetServerSideProps } from 'next';
import { DaprClient, HttpMethod } from '@dapr/dapr';

export const getServerSideProps: GetServerSideProps = async () => {
  const client = new DaprClient();
  const products = await client.invoker.invoke(
    'product-service', 'products', HttpMethod.GET
  );
  return { props: { products } };
};

export default function ProductsPage({ products }: { products: any[] }) {
  return (
    <ul>
      {products.map(p => <li key={p.id}>{p.name} - ${p.price}</li>)}
    </ul>
  );
}
```

## Running with Dapr CLI

```bash
dapr run \
  --app-id shop-frontend \
  --app-port 3000 \
  --dapr-http-port 3500 \
  -- npm run dev
```

## Summary

Next.js with Dapr service invocation uses API routes and `getServerSideProps` for server-side calls to downstream microservices, avoiding browser CORS limitations. The Dapr Node.js SDK provides a clean interface for service invocation, state management, and pub/sub within Next.js API routes.
