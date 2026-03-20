# How to Validate IPv4 Addresses Using Regex in JavaScript

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: JavaScript, Regex, IPv4, Validation, Networking, Node.js

Description: Learn how to validate IPv4 address strings using regular expressions in JavaScript, with patterns that correctly enforce the 0-255 octet range.

## Simple Pattern (Not Recommended)

```javascript
// Naive: matches format but not octet ranges
const naiveIPv4 = /^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$/;

console.log(naiveIPv4.test("192.168.1.1"));    // true
console.log(naiveIPv4.test("999.999.999.999")); // true (WRONG! accepts invalid)
```

## Strict Regex (Validates 0-255)

```javascript
// Each octet matches 0-255 without leading zeros (except "0" itself)
const octet = '(?:25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]\\d|\\d)';
const strictIPv4 = new RegExp(`^${octet}\\.${octet}\\.${octet}\\.${octet}$`);

function isValidIPv4(s) {
    if (typeof s !== 'string') return false;
    return strictIPv4.test(s);
}

const tests = [
    ['192.168.1.1', true],
    ['0.0.0.0', true],
    ['255.255.255.255', true],
    ['256.0.0.1', false],       // Octet > 255
    ['192.168.1', false],       // Missing octet
    ['192.168.1.1.1', false],   // Extra octet
    ['192.168.01.1', false],    // Leading zero
    ['::1', false],             // IPv6
    ['', false],                // Empty
    [' 192.168.1.1', false],    // Leading space
];

tests.forEach(([ip, expected]) => {
    const result = isValidIPv4(ip);
    const status = result === expected ? 'PASS' : 'FAIL';
    console.log(`[${status}] "${ip}" -> ${result}`);
});
```

## Using the Regex as a String Constant

```javascript
// Export as a reusable pattern for use in forms and APIs
const IPV4_REGEX_STR =
    '^(?:25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]\\d|\\d)' +
    '(?:\\.(?:25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]\\d|\\d)){3}$';

module.exports = {
    IPV4_REGEX: new RegExp(IPV4_REGEX_STR),
    isValidIPv4: (s) => new RegExp(IPV4_REGEX_STR).test(s),
};
```

## Extracting IPs from Text (Log Parsing)

```javascript
// Use a non-anchored pattern to find IPs within larger text
const octet = '(?:25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]\\d|\\d)';
const findIPv4 = new RegExp(`\\b${octet}(?:\\.${octet}){3}\\b`, 'g');

function extractIPs(text) {
    return text.match(findIPv4) || [];
}

const logLine = '[2026-03-20] Request from 192.168.1.50 forwarded to 10.0.0.1';
const ips = extractIPs(logLine);
console.log('Found IPs:', ips);  // ['192.168.1.50', '10.0.0.1']
```

## HTML Form Validation with Pattern Attribute

```html
<!-- Use the regex directly in an HTML input pattern attribute -->
<input
  type="text"
  name="ip"
  placeholder="192.168.1.1"
  pattern="^(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}$"
  title="Enter a valid IPv4 address"
  required
/>
```

## TypeScript Version

```typescript
const IPV4_REGEX = /^(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}$/;

function isValidIPv4(s: string): boolean {
    return IPV4_REGEX.test(s);
}

// With custom error
function parseIPv4(s: string): string {
    if (!isValidIPv4(s)) {
        throw new Error(`Invalid IPv4 address: ${s}`);
    }
    return s;
}
```

## Conclusion

A strict JavaScript IPv4 regex must encode the 0-255 octet range with `(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)` for each of the four octets. Always compile the regex once (outside functions) and reuse it. For log parsing, use the pattern without `^$` anchors and add the `g` flag. On the server side (Node.js), consider the `net` module's `net.isIPv4()` function as a simpler alternative.
