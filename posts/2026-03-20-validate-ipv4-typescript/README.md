# How to Validate IPv4 Addresses in TypeScript

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TypeScript, IPv4, Validation, Regex, Networking, Node.js

Description: Learn how to validate IPv4 address strings in TypeScript using regex, type guards, and branded types, with patterns that correctly enforce the 0-255 octet range.

## Strict Regex Validator

```typescript
// Compile once at module level - RegExp is reusable
const IPV4_REGEX =
    /^(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}$/;

export function isValidIPv4(s: string): boolean {
    return IPV4_REGEX.test(s);
}

// Test suite
const tests: [string, boolean][] = [
    ["192.168.1.1",     true],
    ["0.0.0.0",         true],
    ["255.255.255.255", true],
    ["256.0.0.1",       false],   // Octet > 255
    ["192.168.1",       false],   // Missing octet
    ["192.168.1.1.1",   false],   // Extra octet
    ["192.168.01.1",    false],   // Leading zero
    ["::1",             false],   // IPv6
    ["",                false],   // Empty
    [" 192.168.1.1",    false],   // Leading space
];

for (const [ip, expected] of tests) {
    const result = isValidIPv4(ip);
    const status = result === expected ? "PASS" : "FAIL";
    console.log(`[${status}] ${ip.padEnd(25)} -> ${result}`);
}
```

## Type Guard for Narrowing

```typescript
// Branded type - opaque string that has been validated
type IPv4Address = string & { readonly __brand: "IPv4Address" };

function isIPv4(s: string): s is IPv4Address {
    return IPV4_REGEX.test(s);
}

function processIP(raw: string): void {
    if (!isIPv4(raw)) {
        throw new TypeError(`Invalid IPv4 address: ${raw}`);
    }
    // TypeScript now knows raw is IPv4Address
    console.log("Valid IP:", raw);
}
```

## Parsing with Error Throwing

```typescript
export function parseIPv4(s: string): IPv4Address {
    if (!IPV4_REGEX.test(s)) {
        throw new Error(`Invalid IPv4 address: "${s}"`);
    }
    return s as IPv4Address;
}

// Usage
try {
    const ip = parseIPv4("192.168.1.1");
    console.log("Parsed:", ip);
} catch (e) {
    console.error(e);
}
```

## Extracting IPs from Log Text

```typescript
const IPV4_FINDER =
    /\b(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}\b/g;

export function extractIPv4s(text: string): string[] {
    return text.match(IPV4_FINDER) ?? [];
}

const log = "[2026-03-20] Request from 192.168.1.50 to 10.0.0.1";
console.log(extractIPv4s(log));  // ["192.168.1.50", "10.0.0.1"]
```

## Zod Schema Integration

```typescript
import { z } from "zod";

const IPv4Schema = z.string().refine(isValidIPv4, {
    message: "Must be a valid IPv4 address",
});

// Usage in a request body schema
const RequestSchema = z.object({
    clientIp:  IPv4Schema,
    serverIp:  IPv4Schema,
    port:      z.number().int().min(1).max(65535),
});

type Request = z.infer<typeof RequestSchema>;
```

## Conclusion

A compiled `RegExp` literal is the most performant approach for repeated validation in TypeScript. Pair the regex with a TypeScript type guard (`s is IPv4Address`) to narrow types and prevent invalid values from flowing through the type system. For runtime validation in API handlers, integrate the regex into a Zod `refine()` or class-validator decorator to get declarative, schema-level enforcement. On Node.js, `require("net").isIPv4(s)` is also available as a zero-dependency alternative.
