# How to Use Redis for Remix Loader Cache

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Remix, Caching

Description: Learn how to use Redis to cache Remix loader data, reducing database queries on repeated navigations and improving server response times.

---

Remix loaders run on the server for every request. For data that changes infrequently - product catalogs, user profiles, configuration - Redis caching eliminates redundant database queries and speeds up every page load.

## Setup Redis in Remix

```typescript
// app/lib/redis.server.ts
import { createClient } from "redis";

let redis: ReturnType<typeof createClient>;

declare global {
  var __redis: ReturnType<typeof createClient> | undefined;
}

if (process.env.NODE_ENV === "production") {
  redis = createClient({ url: process.env.REDIS_URL });
  redis.connect();
} else {
  // Reuse connection in dev to avoid too many connections
  if (!global.__redis) {
    global.__redis = createClient({ url: process.env.REDIS_URL });
    global.__redis.connect();
  }
  redis = global.__redis;
}

export { redis };
```

## Cache a Loader with Wrapper Function

```typescript
// app/lib/cached-loader.server.ts
import { redis } from "./redis.server";

export async function cachedLoader<T>(
  cacheKey: string,
  ttl: number,
  fetchFn: () => Promise<T>
): Promise<T> {
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached) as T;
  }

  const data = await fetchFn();
  await redis.set(cacheKey, JSON.stringify(data), { EX: ttl });
  return data;
}
```

## Use in a Remix Loader

```typescript
// app/routes/products.$id.tsx
import { json, LoaderFunctionArgs } from "@remix-run/node";
import { useLoaderData } from "@remix-run/react";
import { cachedLoader } from "~/lib/cached-loader.server";
import { db } from "~/lib/db.server";

export async function loader({ params }: LoaderFunctionArgs) {
  const { id } = params;

  const product = await cachedLoader(
    `product:${id}`,
    300, // 5 minute TTL
    () => db.product.findUniqueOrThrow({ where: { id } })
  );

  return json({ product });
}

export default function ProductPage() {
  const { product } = useLoaderData<typeof loader>();
  return <h1>{product.title}</h1>;
}
```

## Cache Authenticated User Data

```typescript
// app/routes/_app.tsx (layout route loader)
export async function loader({ request }: LoaderFunctionArgs) {
  const session = await getSession(request.headers.get("Cookie"));
  const userId = session.get("userId");

  if (!userId) {
    throw redirect("/login");
  }

  const user = await cachedLoader(
    `user:profile:${userId}`,
    120, // 2 minute TTL for user data
    () => db.user.findUniqueOrThrow({ where: { id: userId } })
  );

  return json({ user });
}
```

## Invalidate Cache from Actions

```typescript
// app/routes/products.$id.edit.tsx
import { redis } from "~/lib/redis.server";

export async function action({ params, request }: ActionFunctionArgs) {
  const { id } = params;
  const formData = await request.formData();

  await db.product.update({
    where: { id },
    data: {
      title: String(formData.get("title")),
      price: Number(formData.get("price")),
    },
  });

  // Invalidate the cache for this product
  await redis.del(`product:${id}`);

  return redirect(`/products/${id}`);
}
```

## Cache Listing Pages with Pagination

```typescript
export async function loader({ request }: LoaderFunctionArgs) {
  const url = new URL(request.url);
  const page = url.searchParams.get("page") || "1";
  const category = url.searchParams.get("category") || "all";

  const cacheKey = `products:list:${category}:page:${page}`;

  const data = await cachedLoader(cacheKey, 60, async () => {
    const skip = (Number(page) - 1) * 20;
    return db.product.findMany({
      where: category !== "all" ? { category } : {},
      skip,
      take: 20,
      orderBy: { createdAt: "desc" },
    });
  });

  return json({ products: data });
}
```

## Summary

Caching Remix loader data in Redis reduces database load for frequently accessed routes without changing the loader's external interface. A thin `cachedLoader` wrapper handles cache lookups and population, and Remix actions are the natural place to invalidate stale entries after mutations. This pattern is especially effective for public pages with many anonymous visitors hitting the same routes.
