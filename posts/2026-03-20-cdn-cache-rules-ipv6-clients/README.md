# How to Configure CDN Cache Rules for IPv6 Clients

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, CDN, Caching, Cache Rules, Cloudflare, Fastly

Description: A guide to configuring CDN cache rules that handle IPv6 clients correctly, including cache key variations, TTL policies, and IPv6-specific caching considerations.

CDN caching for IPv6 clients generally works the same as IPv4 - the cache key is typically the URL, not the client IP. However, there are specific scenarios where IPv6 client information affects caching behavior.

## Do IPv4 and IPv6 Clients Get Different Cached Content?

By default: **No**. CDN cache keys are based on URL, headers (like Accept-Encoding), and other request attributes - not the client IP version. Both IPv4 and IPv6 clients receive the same cached response.

Exceptions where IP version might matter:
- Geo-based content differentiation (IPv6 and IPv4 may geolocate differently)
- A/B testing by IP range
- Content that includes the client's IP address

## Cloudflare Cache Rules

```javascript
// Cloudflare Workers: Custom cache key including IP version
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const clientIP = request.headers.get('CF-Connecting-IP');
  const isIPv6 = clientIP && clientIP.includes(':');

  // Most content: don't vary by IP version
  const cacheKey = new Request(request.url, {
    method: request.method,
    headers: {
      'Accept-Language': request.headers.get('Accept-Language') || '',
      'Accept-Encoding': request.headers.get('Accept-Encoding') || '',
    }
  });

  const cache = caches.default;
  let response = await cache.match(cacheKey);

  if (!response) {
    response = await fetch(request);

    // Cache for both IPv4 and IPv6 clients
    if (response.ok) {
      const modifiedResponse = new Response(response.body, response);
      modifiedResponse.headers.set('Cache-Control', 'max-age=3600');
      event.waitUntil(cache.put(cacheKey, modifiedResponse.clone()));
    }
  }

  return response;
}
```

## Cloudflare Page Rules for IPv6

```text
# Cloudflare Page Rule: Cache everything for CDN assets

URL pattern: cdn.example.com/*
Settings:
  Cache Level: Cache Everything
  Edge Cache TTL: 1 month

# This applies equally to IPv4 and IPv6 clients
```

## Fastly VCL: Cache Key for IPv6

```vcl
sub vcl_hash {
  hash_data(req.url);
  hash_data(req.http.Host);

  # For most content: don't include IP version in cache key
  # Both IPv4 and IPv6 clients share the same cache

  # Exception: if content varies by geographic region and IPv4/IPv6
  # geolocate differently, add region to cache key:
  # if (client.geo.country_code) {
  #   hash_data(client.geo.country_code);
  # }
}
```

## Nginx Cache Configuration

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=ipv6_cache:10m max_size=10g;

server {
    listen [::]:443 ssl;

    location / {
        proxy_cache ipv6_cache;

        # Cache key: by default doesn't include client IP
        proxy_cache_key "$host$request_uri";

        # Both IPv4 and IPv6 clients use same cached content
        proxy_cache_valid 200 1h;
        proxy_cache_valid 404 1m;

        # Add X-Cache-Status header for debugging
        add_header X-Cache-Status $upstream_cache_status;

        proxy_pass http://backends;
    }
}
```

## When to Vary Cache by IPv6

### Geo-Based Content

```javascript
// Cloudflare Worker: Vary by country (affects IPv6 geolocation)
async function handleRequest(request) {
  const country = request.cf.country || 'US';
  const url = new URL(request.url);
  url.searchParams.set('_cf_country', country);

  // Different IPv6 prefix ranges may geolocate to different countries
  const cacheKey = new Request(url.toString());
  return fetch(cacheKey);
}
```

### IPv6-Specific Content

```nginx
# If your application serves different content to IPv6 clients:
proxy_cache_key "$host$request_uri$http_x_client_ip_version";

# Pass IP version to backend
set $ip_version "IPv4";
if ($remote_addr ~ ":") {
    set $ip_version "IPv6";
}
proxy_set_header X-Client-IP-Version $ip_version;
```

## Cache Purging for IPv6

Cache purging works the same regardless of client IP version - purge by URL:

```bash
# Cloudflare: Purge specific URL (affects both IPv4 and IPv6 cached versions)
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"files": ["https://example.com/path/to/resource"]}'

# Fastly: Purge by URL
curl -X PURGE https://cdn.example.com/path/to/resource \
  -H "Fastly-Key: $FASTLY_API_KEY"
```

## Monitoring Cache Hit Rate for IPv6 Clients

```promql
# If your CDN exports metrics by client IP version:
sum(rate(cdn_requests_total{ip_version="ipv6", cache_status="HIT"}[5m])) /
sum(rate(cdn_requests_total{ip_version="ipv6"}[5m]))
```

IPv6 clients typically achieve similar cache hit rates as IPv4 clients since CDN caching is URL-based - the only exception is when content genuinely varies by client network characteristics that differ between IPv4 and IPv6 deployments.
