# How to Use Redis for Next.js Rate Limiting at the Edge

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Next.js, Rate Limiting, Edge Middleware, Upstash

Description: Rate limit Next.js requests at the edge using Upstash Redis and Next.js Middleware to block abusive traffic before it reaches your application.

---

Next.js Middleware runs at the Edge runtime before requests reach your pages or API routes. Pairing it with Upstash Redis (an HTTP-based Redis service compatible with the Edge runtime) enables sub-millisecond rate limiting globally.

## Why Edge Rate Limiting

- Runs before your serverless functions, saving compute costs
- Enforces limits at Vercel's edge nodes closest to the user
- Upstash Redis uses HTTP/REST, which works in the Edge runtime where Node.js TCP is unavailable

## Install Upstash SDK

```bash
npm install @upstash/ratelimit @upstash/redis
```

## Configure Rate Limit

`middleware.ts`:

```typescript
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(), // UPSTASH_REDIS_REST_URL and UPSTASH_REDIS_REST_TOKEN
  limiter: Ratelimit.slidingWindow(60, "1 m"),
  analytics: true,
  prefix: "myapp:rl",
});

export async function middleware(request: NextRequest) {
  // Only rate limit API routes
  if (!request.nextUrl.pathname.startsWith("/api/")) {
    return NextResponse.next();
  }

  const identifier = request.ip ?? request.headers.get("x-forwarded-for") ?? "anonymous";
  const { success, limit, reset, remaining } = await ratelimit.limit(identifier);

  if (!success) {
    return new NextResponse(
      JSON.stringify({ error: "Too many requests" }),
      {
        status: 429,
        headers: {
          "Content-Type": "application/json",
          "X-RateLimit-Limit": limit.toString(),
          "X-RateLimit-Remaining": remaining.toString(),
          "X-RateLimit-Reset": reset.toString(),
        },
      }
    );
  }

  const response = NextResponse.next();
  response.headers.set("X-RateLimit-Limit", limit.toString());
  response.headers.set("X-RateLimit-Remaining", remaining.toString());
  return response;
}

export const config = {
  matcher: "/api/:path*",
};
```

## Environment Variables

```text
UPSTASH_REDIS_REST_URL=https://xxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=your-token-here
```

## Different Limits per Route

```typescript
const strictLimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.fixedWindow(10, "1 m"),
});

const normalLimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(100, "1 m"),
});

export async function middleware(request: NextRequest) {
  const path = request.nextUrl.pathname;
  const limiter = path.startsWith("/api/auth") ? strictLimit : normalLimit;
  const { success } = await limiter.limit(request.ip ?? "anon");
  if (!success) return new NextResponse("Rate limited", { status: 429 });
  return NextResponse.next();
}
```

## Test Rate Limiting

```bash
for i in $(seq 1 65); do
  curl -s -o /dev/null -w "%{http_code}\n" https://myapp.vercel.app/api/data
done
```

## Summary

Next.js Edge Middleware with Upstash Redis enforces rate limits globally before any serverless function is invoked, reducing costs and protecting against abuse. The sliding window algorithm prevents burst attacks at window boundaries, and separate limits for sensitive routes like `/api/auth` add an extra layer of protection against credential stuffing.
