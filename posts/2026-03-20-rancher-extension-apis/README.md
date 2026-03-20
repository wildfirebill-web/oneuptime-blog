# How to Use Rancher Extension APIs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Extensions, API, UI

Description: Learn how to use Rancher Extension APIs to build custom UI components, register routes, and integrate with Rancher's core services.

## Introduction

Rancher's Extension API (`@rancher/shell`) provides a rich set of hooks and utilities that let extension developers register custom pages, panels, actions, and resource detail tabs directly into the Rancher dashboard. This guide covers the most important API surfaces and how to use them effectively.

## Prerequisites

- A scaffolded Rancher Extension project (`yarn create @rancher/shell my-ext`)
- Familiarity with Vue 3 and the Composition API
- Node.js 16+ and Yarn

## Extension Entry Point

Every extension registers itself through an `index.js` (or `index.ts`) that exports a default function receiving the `$plugin` context:

```javascript
// pkg/my-extension/index.js

// This function is called when Rancher loads the extension
export default function(context) {
  const { $plugin } = context;

  // Register a top-level navigation item
  $plugin.addNavItem({
    label:   'My Extension',
    icon:    'icon-extension',
    route:   { name: 'my-ext-home' },
  });

  // Register routes for the extension's pages
  $plugin.addRoutes([
    {
      name:      'my-ext-home',
      path:      '/my-ext',
      component: () => import('./pages/Home.vue'),
    },
  ]);
}
```

## Registering Resource Detail Tabs

Add a custom tab to any resource detail page (e.g., Deployments):

```javascript
// Add a tab to the Deployment detail view
$plugin.addTab({
  name:       'my-metrics-tab',
  label:      'Custom Metrics',
  resource:   'apps.deployment',   // <group>.<kind>
  component:  () => import('./tabs/MetricsTab.vue'),
  weight:     100,                 // Controls tab order (higher = later)
});
```

```vue
<!-- tabs/MetricsTab.vue -->
<template>
  <div>
    <h2>Custom Metrics for {{ resource.metadata.name }}</h2>
    <!-- Render your charts here -->
  </div>
</template>

<script setup>
// The `resource` prop is automatically injected with the current resource object
defineProps({ resource: Object });
</script>
```

## Registering Action Buttons

Inject custom action buttons into resource list and detail pages:

```javascript
// Add a bulk action to the Pod list view
$plugin.addAction({
  label:     'Restart Selected',
  icon:      'icon-refresh',
  resource:  'v1.pod',
  multiple:  true,   // Show in bulk-action bar
  handler(resources) {
    resources.forEach(pod => {
      // Call your custom restart logic
      restartPod(pod);
    });
  },
});
```

## Using the Store

Extensions have access to Rancher's Vuex store through the `useStore` composable:

```javascript
// Fetch all deployments from the management cluster
import { useStore } from '@shell/composables/store';

const store = useStore();

// Dispatch a find-all request
const deployments = await store.dispatch('cluster/findAll', {
  type: 'apps.deployment',
});
```

## Making API Requests

Use the built-in HTTP client to interact with the Rancher or Kubernetes API:

```javascript
import { useStore } from '@shell/composables/store';

const store = useStore();

// GET request to the Rancher v3 API
const response = await store.dispatch('management/request', {
  method: 'GET',
  url:    '/v3/clusters',
});

// GET request to a specific cluster's Kubernetes API
const pods = await store.dispatch('cluster/request', {
  method: 'GET',
  url:    `/api/v1/namespaces/default/pods`,
  clusterId: 'c-xxxxx',
});
```

## Registering Dashboard Panels

Add a widget to the Rancher home dashboard:

```javascript
$plugin.addPanel({
  name:      'my-summary-panel',
  label:     'My Summary',
  component: () => import('./panels/SummaryPanel.vue'),
  weight:    50,
});
```

## Listening to Extension Events

Extensions can subscribe to lifecycle events:

```javascript
// Called when the current cluster changes
$plugin.on('cluster-changed', (clusterId) => {
  console.log('User switched to cluster:', clusterId);
  // Refresh your data here
});
```

## Conclusion

The Rancher Extension API provides all the building blocks needed to deeply integrate custom functionality into the Rancher dashboard. By leveraging route registration, resource tabs, action buttons, store access, and panel APIs, you can build powerful extensions that feel like native parts of the platform. Always consult the `@rancher/shell` source and the official Rancher Extension documentation for the latest API signatures.
