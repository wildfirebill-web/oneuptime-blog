# How to Use Upstash Redis with Edge Functions (Vercel, Cloudflare)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Upstash, Edge Function, Vercel, Cloudflare, Serverless

Description: Learn how to connect Upstash Redis to Vercel Edge Functions and Cloudflare Workers using the HTTP-based REST API for low-latency serverless caching.

---

Edge functions run close to the user, but they have strict constraints: no persistent TCP connections, limited runtime APIs, and tight execution budgets. Traditional Redis clients require a long-lived TCP connection, which is incompatible with this model. Upstash Redis solves this by providing an HTTP REST API that works anywhere a `fetch` call can be made - including Vercel Edge Middleware and Cloudflare Workers.

## How Upstash Redis Works

Upstash exposes a serverless Redis instance accessible via HTTPS. Each command maps to a REST endpoint:

```text
POST https://<UPSTASH_REDIS_REST_URL>/set/mykey/myvalue
Authorization: Bearer <UPSTASH_REDIS_REST_TOKEN>
```

The `@upstash/redis` SDK abstracts this into familiar Redis commands while using `fetch` under the hood.

## Setting Up Upstash

1. Create a free account at [upstash.com](https://upstash.com).
2. Create a new Redis database in your preferred region.
3. Copy `UPSTASH_REDIS_REST_URL` and `UPSTASH_REDIS_REST_TOKEN` from the console.

## Using Upstash with Vercel Edge Middleware

Install the SDK:

```bash
npm install @upstash/redis
```

Create `middleware.ts` at the project root:

```typescript
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";
import { Redis } from "@upstash/redis";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

export async function middleware(request: NextRequest) {
  const ip = request.ip ?? "anonymous";
  const key = `rate-limit:${ip}`;

  const requests = await redis.incr(key);
  if (requests === 1) {
    await redis.expire(key, 60); // 1-minute window
  }

  if (requests > 30) {
    return new NextResponse("Too Many Requests", { status: 429 });
  }

  return NextResponse.next();
}

export const config = {
  matcher: "/api/:path*",
};
```

Add environment variables in your Vercel project dashboard or `.env.local`:

```bash
UPSTASH_REDIS_REST_URL=https://your-db.upstash.io
UPSTASH_REDIS_REST_TOKEN=your-token-here
```

## Using Upstash with Cloudflare Workers

Cloudflare Workers use a slightly different module format. Bind the environment variables in `wrangler.toml`:

```toml
[vars]
UPSTASH_REDIS_REST_URL = "https://your-db.upstash.io"
UPSTASH_REDIS_REST_TOKEN = "your-token-here"
```

Then use the SDK in your Worker:

```typescript
import { Redis } from "@upstash/redis/cloudflare";

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const redis = new Redis({
      url: env.UPSTASH_REDIS_REST_URL,
      token: env.UPSTASH_REDIS_REST_TOKEN,
    });

    const cached = await redis.get<string>("homepage");
    if (cached) {
      return new Response(cached, {
        headers: { "X-Cache": "HIT" },
      });
    }

    const fresh = await fetch("https://api.example.com/homepage").then((r) =>
      r.text()
    );
    await redis.set("homepage", fresh, { ex: 300 }); // cache 5 minutes

    return new Response(fresh, { headers: { "X-Cache": "MISS" } });
  },
};
```

## Key Considerations

- **Latency**: Upstash global replication keeps reads within 10-20ms of edge nodes.
- **Cold starts**: No connection setup overhead - each request is a fresh HTTP call.
- **Pricing**: Upstash charges per request, so batch reads with `mget` or pipelines where possible.
- **Pipelines**: Use `redis.pipeline()` to batch multiple commands in a single HTTP round trip.

```typescript
const pipeline = redis.pipeline();
pipeline.get("key1");
pipeline.get("key2");
pipeline.incr("counter");
const [val1, val2, counter] = await pipeline.exec();
```

## Summary

Upstash Redis bridges the gap between edge runtimes and Redis by replacing TCP connections with HTTP fetch calls. You can implement rate limiting, caching, and session storage at the edge on both Vercel and Cloudflare Workers with minimal setup. Use pipeline commands to keep per-request costs and latency low.
