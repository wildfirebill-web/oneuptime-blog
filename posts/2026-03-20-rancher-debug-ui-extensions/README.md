# How to Debug Rancher UI Extensions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Extensions, Debugging, UI

Description: A practical guide to debugging Rancher UI extensions using local development mode, browser DevTools, and common troubleshooting techniques.

## Introduction

Debugging Rancher UI Extensions requires a different approach than traditional web development because your code runs inside the Rancher shell at runtime. This guide walks through setting up a local development environment, using browser DevTools effectively, and resolving the most common issues extension developers encounter.

## Prerequisites

- Rancher running (locally via Docker or in a cluster)
- Extension project scaffolded with `@rancher/shell`
- Node.js 16+ and Yarn
- Chrome or Firefox DevTools familiarity

## Step 1: Run in Local Development Mode

The fastest way to debug is to serve your extension locally against a live Rancher instance:

```bash
# Start the local dev server - this proxy your extension into a running Rancher

API=https://<rancher-url> yarn dev

# The Rancher UI will be available at http://localhost:8005
# Your extension code is hot-reloaded on every save
```

The `API` environment variable tells the dev server where to proxy API calls. Your browser connects to `localhost:8005` but communicates with the real Rancher backend.

## Step 2: Enable Vue DevTools

Install the [Vue DevTools](https://devtools.vuejs.org/) browser extension. Once installed:

1. Open DevTools (`F12`).
2. Navigate to the **Vue** tab.
3. Inspect component trees, props, and computed values in real time.

```javascript
// Temporarily expose the Vue app instance for debugging
// Add this to your component during development ONLY
onMounted(() => {
  window.__MY_EXT_DEBUG__ = { store: useStore(), route: useRoute() };
  console.log('Debug context available at window.__MY_EXT_DEBUG__');
});
```

## Step 3: Inspect the Rancher Store

The Rancher Vuex store holds all resource data. You can inspect it from the browser console:

```javascript
// Access the global Nuxt/Vue app instance
const app = window.__vue_app__;

// Get a reference to the store
const store = app.config.globalProperties.$store;

// List all loaded clusters
console.log(store.getters['management/all']('provisioning.cattle.io.cluster'));

// Check the current cluster ID
console.log(store.getters['currentCluster']?.id);
```

## Step 4: Debug API Requests

Use the Network tab in DevTools to inspect API calls:

1. Open DevTools → **Network** tab.
2. Filter by `Fetch/XHR`.
3. Trigger an action in your extension.
4. Click the request to see headers, payload, and response.

For programmatic debugging, wrap your store dispatches:

```javascript
// Wrap dispatch calls with logging during development
async function debugDispatch(store, action, payload) {
  console.group(`[dispatch] ${action}`);
  console.log('Payload:', payload);
  try {
    const result = await store.dispatch(action, payload);
    console.log('Result:', result);
    return result;
  } catch (err) {
    console.error('Error:', err);
    throw err;
  } finally {
    console.groupEnd();
  }
}
```

## Step 5: Debug Extension Registration

If your extension routes or tabs aren't appearing, check the registration:

```javascript
// In your extension's index.js, log registration steps
export default function({ $plugin }) {
  console.log('[my-extension] Registering routes...');

  $plugin.addRoutes([
    {
      name: 'my-ext-home',
      path: '/my-ext',
      component: () => {
        console.log('[my-extension] Loading Home component');
        return import('./pages/Home.vue');
      },
    },
  ]);

  console.log('[my-extension] Registration complete');
}
```

## Step 6: Check for Common Errors

### Extension Not Loading

```bash
# Check that the extension chart is installed
kubectl get helmchart -n cattle-system

# Check extension pod logs
kubectl logs -n cattle-system -l app=my-extension --tail=100
```

### Vue Component Errors

Look for errors in the browser console. Common causes:

- **`Cannot read properties of undefined`** - A store getter returned `undefined` before data was loaded. Use `computed()` and guard with `?.`.
- **`[Vue warn]: Missing required prop`** - The parent component isn't passing required props.
- **`NavigationDuplicated`** - You're navigating to the current route. Guard with `if (route.name !== targetName)`.

### CORS Errors in Dev Mode

```bash
# If you see CORS errors, ensure your Rancher URL uses HTTPS
# and that the dev server proxy is configured correctly
API=https://rancher.example.com yarn dev --open
```

## Step 7: Write Unit Tests for Extension Logic

```javascript
// tests/unit/my-logic.spec.js
import { describe, it, expect } from 'vitest';
import { formatMetric } from '../utils/metrics';

describe('formatMetric', () => {
  it('formats bytes to human-readable strings', () => {
    expect(formatMetric(1024)).toBe('1 KiB');
    expect(formatMetric(1048576)).toBe('1 MiB');
  });
});
```

```bash
# Run unit tests
yarn test:unit
```

## Conclusion

Debugging Rancher UI Extensions becomes manageable once you have the local dev server running, Vue DevTools installed, and a solid understanding of the Rancher store. By combining hot-reloading, console inspection, Network tab analysis, and targeted unit tests, you can quickly identify and resolve issues in your extension code.
