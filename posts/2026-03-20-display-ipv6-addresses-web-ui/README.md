# How to Display IPv6 Addresses in Web UI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, UI, JavaScript, React, Formatting, Display, Bracket Notation

Description: Format and display IPv6 addresses correctly in web interfaces, including bracket notation for URLs, shortened display forms, and copy-to-clipboard functionality.

## Introduction

IPv6 addresses in web UIs require special handling: they need bracket notation in URLs, can be long and hard to read, and users may need to copy them in different formats. This post covers formatting helpers, React components, and UX patterns for displaying IPv6 addresses.

## Step 1: JavaScript Formatting Utilities

```javascript
// utils/ipv6.js

/**
 * Format an IPv6 address for display in URLs (adds brackets).
 * IPv4 addresses are returned unchanged.
 */
export function formatForURL(address) {
    // Check if it's already bracketed
    if (address.startsWith('[') && address.endsWith(']')) {
        return address;
    }
    // Simple heuristic: IPv6 contains colons
    if (address.includes(':')) {
        return `[${address}]`;
    }
    return address;  // IPv4
}

/**
 * Truncate a long IPv6 address for compact display.
 * Shows first and last group with ellipsis.
 */
export function truncateIPv6(address, maxLength = 20) {
    if (address.length <= maxLength) return address;
    const half = Math.floor(maxLength / 2) - 1;
    return `${address.slice(0, half)}…${address.slice(-half)}`;
}

/**
 * Expand compressed IPv6 to full form for display.
 * Useful when showing the full address for debugging.
 */
export function expandIPv6(address) {
    // Use URL API to normalize
    try {
        const url = new URL(`http://[${address}]/`);
        const host = url.hostname; // [full::form]
        return host.slice(1, -1);  // Remove brackets
    } catch {
        return address;
    }
}

/**
 * Get the /64 prefix for display.
 */
export function getIPv6Prefix64(address) {
    const parts = address.split(':');
    if (parts.length >= 4) {
        return parts.slice(0, 4).join(':') + '::/64';
    }
    return address;
}
```

## Step 2: React IPv6 Display Component

```jsx
// components/IPv6Address.jsx
import React, { useState } from 'react';

export function IPv6Address({ address, showFull = false, copyable = true }) {
    const [copied, setCopied] = useState(false);

    const isIPv6 = address && address.includes(':');
    const displayAddress = showFull ? address : address;

    const copyToClipboard = async () => {
        await navigator.clipboard.writeText(address);
        setCopied(true);
        setTimeout(() => setCopied(false), 2000);
    };

    if (!isIPv6) {
        return <code className="ip-address ipv4">{address}</code>;
    }

    return (
        <span className="ip-address-wrapper">
            <code
                className="ip-address ipv6"
                title={`Full: ${address}`}
                style={{ fontFamily: 'monospace' }}
            >
                {address}
            </code>
            {copyable && (
                <button
                    onClick={copyToClipboard}
                    className="copy-btn"
                    title="Copy IPv6 address"
                    aria-label={`Copy ${address}`}
                >
                    {copied ? '✓ Copied' : 'Copy'}
                </button>
            )}
        </span>
    );
}

// Usage
function ConnectionList({ connections }) {
    return (
        <table>
            <thead>
                <tr>
                    <th>Client IP</th>
                    <th>Connected At</th>
                </tr>
            </thead>
            <tbody>
                {connections.map(conn => (
                    <tr key={conn.id}>
                        <td>
                            <IPv6Address
                                address={conn.ip}
                                copyable={true}
                            />
                        </td>
                        <td>{conn.timestamp}</td>
                    </tr>
                ))}
            </tbody>
        </table>
    );
}
```

## Step 3: Clickable IPv6 Links

```javascript
// utils/createIPv6Link.js

export function createIPv6Link(address, port, protocol = 'http') {
    const isIPv6 = address.includes(':');
    const host = isIPv6 ? `[${address}]` : address;
    if (port) {
        return `${protocol}://${host}:${port}`;
    }
    return `${protocol}://${host}`;
}

// React link component
export function IPv6Link({ address, port, protocol = 'https', children }) {
    const href = createIPv6Link(address, port, protocol);
    return (
        <a href={href} target="_blank" rel="noopener noreferrer">
            {children || `${protocol}://[${address}]${port ? `:${port}` : ''}`}
        </a>
    );
}
```

## Step 4: CSS Styling for IPv6 Addresses

```css
/* ipv6.css */

.ip-address {
    font-family: 'Courier New', Courier, monospace;
    font-size: 0.875rem;
    background: #f0f4f8;
    padding: 2px 6px;
    border-radius: 3px;
    border: 1px solid #d0d7de;
    white-space: nowrap;
    user-select: all;  /* Select all on click */
}

.ip-address.ipv6 {
    color: #1a56db;  /* Blue for IPv6 */
}

.ip-address.ipv4 {
    color: #0e7a0d;  /* Green for IPv4 */
}

.ip-address-wrapper {
    display: inline-flex;
    align-items: center;
    gap: 4px;
}

.copy-btn {
    font-size: 0.75rem;
    padding: 2px 6px;
    cursor: pointer;
    border: 1px solid #d0d7de;
    border-radius: 3px;
    background: white;
}
```

## Step 5: Sorting and Filtering IPv6 in Tables

```javascript
// Sort IPv6 addresses in a table
function sortIPv6(addresses) {
    return addresses.sort((a, b) => {
        // Convert to sortable form using URL normalization
        const normalizeForSort = (ip) => {
            try {
                // Pad each group to 4 hex digits for lexicographic sort
                const url = new URL(`http://[${ip}]/`);
                return url.hostname;
            } catch {
                return ip;
            }
        };
        return normalizeForSort(a).localeCompare(normalizeForSort(b));
    });
}
```

## Conclusion

Displaying IPv6 addresses in web UIs requires bracket notation in hyperlinks, monospace fonts for readability, copy-to-clipboard functionality, and `user-select: all` CSS for easy selection. React components can handle the formatting logic centrally. Use `createIPv6Link()` to generate correct `href` attributes. Monitor your web UI's IPv6 display components with OneUptime's visual regression checks.
