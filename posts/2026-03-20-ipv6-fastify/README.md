# How to Use IPv6 with Fastify

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node.js, IPv6, Fastify, HTTP, REST API, Networking

Description: Build IPv6-capable Fastify applications including binding, client IP extraction, hooks, and schema validation for IPv6 addresses.

## Binding Fastify to IPv6

```javascript
const Fastify = require('fastify');

const app = Fastify({ logger: true });

app.get('/', async (request, reply) => {
    return { message: 'Hello from IPv6 Fastify!' };
});

// Start listening on [::]:3000
app.listen({ port: 3000, host: '::' }, (err, address) => {
    if (err) {
        app.log.error(err);
        process.exit(1);
    }
    app.log.info(`Server on ${address}`);
});
```

## Extracting Client IPv6 Address

Fastify populates `request.ip` with the client address. When behind a proxy, configure `trustProxy`:

```javascript
const Fastify = require('fastify');

const app = Fastify({
    logger: true,
    trustProxy: true,  // Use X-Forwarded-For header
});

// Hook to normalize IP on every request
app.addHook('onRequest', async (request, reply) => {
    let ip = request.ip;

    // Unwrap IPv4-mapped addresses
    if (ip && ip.startsWith('::ffff:')) {
        ip = ip.slice(7);
    }

    request.clientIP = ip;
    request.isIPv6 = ip && ip.includes(':');
});

app.get('/info', async (request) => {
    return {
        ip: request.clientIP,
        version: request.isIPv6 ? 'IPv6' : 'IPv4',
        userAgent: request.headers['user-agent'],
    };
});

app.listen({ port: 3000, host: '::' });
```

## Schema Validation for IPv6

Fastify uses JSON Schema for request validation. Use `format: 'ipv6'` for address fields:

```javascript
const Fastify = require('fastify');
const addFormats = require('ajv-formats');

const app = Fastify({
    ajv: {
        customOptions: {
            strict: false,
        },
        plugins: [addFormats],
    },
});

const deviceSchema = {
    body: {
        type: 'object',
        required: ['name', 'ipv6Address'],
        properties: {
            name: { type: 'string', minLength: 1 },
            ipv6Address: {
                type: 'string',
                format: 'ipv6',
                description: 'IPv6 address of the device',
            },
            prefixLen: {
                type: 'integer',
                minimum: 0,
                maximum: 128,
            },
        },
    },
    response: {
        201: {
            type: 'object',
            properties: {
                id: { type: 'string' },
                name: { type: 'string' },
                ipv6Address: { type: 'string' },
            },
        },
    },
};

const devices = new Map();
let idCounter = 0;

app.post('/devices', { schema: deviceSchema }, async (request, reply) => {
    const { name, ipv6Address } = request.body;
    const id = String(++idCounter);
    devices.set(id, { id, name, ipv6Address });
    return reply.status(201).send({ id, name, ipv6Address });
});

app.listen({ port: 3000, host: '::' });
```

## Plugin for IPv6 Rate Limiting

```javascript
const Fastify = require('fastify');
const fp = require('fastify-plugin');

// Rate limit plugin grouped by /64 prefix
async function ipv6RateLimitPlugin(fastify, options) {
    const { limit = 100, windowMs = 60_000 } = options;
    const buckets = new Map();

    function getKey(ip) {
        if (ip && ip.includes(':')) {
            return ip.split(':').slice(0, 4).join(':') + '::/64';
        }
        return ip || 'unknown';
    }

    fastify.addHook('onRequest', async (request, reply) => {
        let ip = request.ip || '';
        if (ip.startsWith('::ffff:')) ip = ip.slice(7);

        const key = getKey(ip);
        const now = Date.now();
        const bucket = buckets.get(key) || { count: 0, reset: now + windowMs };

        if (now > bucket.reset) {
            bucket.count = 0;
            bucket.reset = now + windowMs;
        }

        bucket.count++;
        buckets.set(key, bucket);

        if (bucket.count > limit) {
            return reply.status(429).send({ error: 'Too Many Requests' });
        }
    });
}

const app = Fastify({ trustProxy: true });
app.register(fp(ipv6RateLimitPlugin), { limit: 100, windowMs: 60_000 });

app.get('/', async () => ({ status: 'ok' }));
app.listen({ port: 3000, host: '::' });
```

## Fastify Lifecycle Logging for IPv6

```javascript
const Fastify = require('fastify');

const app = Fastify({
    logger: {
        serializers: {
            req(request) {
                let ip = request.socket?.remoteAddress || '';
                if (ip.startsWith('::ffff:')) ip = ip.slice(7);
                return {
                    method: request.method,
                    url: request.url,
                    ip,
                    isIPv6: ip.includes(':'),
                };
            },
        },
    },
});

app.get('/', async (request, reply) => {
    app.log.info({ ip: request.ip }, 'Handling request');
    return { ok: true };
});

app.listen({ port: 3000, host: '::' });
```

## Conclusion

Fastify supports IPv6 via `host: '::'` in the listen options. Set `trustProxy: true` to use `X-Forwarded-For` when deployed behind a proxy. JSON Schema validation with `format: 'ipv6'` (via `ajv-formats`) handles IPv6 address validation in request bodies automatically. Hooks are the idiomatic Fastify pattern for cross-cutting concerns like IP normalization and rate limiting. Rate-limit by `/64` prefix to handle IPv6 privacy extensions where clients rotate individual addresses.
