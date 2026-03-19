# How to Build Custom Rancher UI Extensions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, UI Extensions, Plugins, Dashboard

Description: Step-by-step guide to building custom UI extensions for the Rancher dashboard, including project setup, component development, and deployment.

Rancher UI Extensions let you add custom pages, resource views, and functionality to the Rancher dashboard without modifying the core codebase. This guide walks you through creating, developing, and deploying a custom UI extension from scratch.

## Prerequisites

You need the following tools installed:

- Node.js 16+ and npm or yarn
- Git
- Access to a Rancher v2.7+ instance with Extensions support enabled

## Setting Up the Development Environment

### Step 1: Install the Rancher Shell Extension Creator

The `@rancher/shell` package provides the tools for creating and building extensions:

```bash
# Create a new extension project
npx @rancher/create-extension my-extension

cd my-extension
```

This scaffolds a project with the following structure:

```
my-extension/
  pkg/
    my-extension/
      index.ts
      product.ts
  package.json
  tsconfig.json
  vue.config.js
```

### Step 2: Install Dependencies

```bash
yarn install
```

### Step 3: Start the Development Server

```bash
yarn dev
```

This starts a local development server that proxies requests to your Rancher instance. Open the URL shown in the terminal and log in with your Rancher credentials.

## Creating a Basic Extension

### Define the Extension Entry Point

Edit `pkg/my-extension/index.ts`:

```typescript
import { importTypes } from '@rancher/auto-import';
import { IPlugin } from '@shell/core/types';

const onEnter = (store: any, config: any) => {
  // Called when the extension is loaded
  console.log('My Extension loaded');
};

const onLeave = (store: any, config: any) => {
  // Called when the extension is unloaded
  console.log('My Extension unloaded');
};

const extension: IPlugin = {
  name: 'my-extension',
  routes: [],
  stores: [],
  onEnter,
  onLeave,
};

export default extension;
```

### Register a Product (Top-Level Menu Item)

Create `pkg/my-extension/product.ts`:

```typescript
import { IPlugin } from '@shell/core/types';

export function init($plugin: IPlugin, store: any) {
  const {
    product,
    basicType,
    virtualType,
  } = $plugin.DSL(store, $plugin.name);

  // Register a new product in the side navigation
  product({
    icon: 'gear',
    inStore: 'management',
    weight: 100,
    to: {
      name: `c-cluster-${$plugin.name}-overview`,
      params: { product: $plugin.name }
    }
  });

  // Register a virtual resource type
  virtualType({
    name: 'overview',
    labelKey: 'Overview',
    route: {
      name: `c-cluster-${$plugin.name}-overview`,
      params: { product: $plugin.name }
    }
  });

  basicType(['overview']);
}
```

### Create a Page Component

Create `pkg/my-extension/pages/overview.vue`:

```vue
<template>
  <div class="overview-page">
    <h1>My Custom Extension</h1>

    <div class="stats-grid">
      <div class="stat-card" v-for="stat in stats" :key="stat.label">
        <h3>{{ stat.value }}</h3>
        <p>{{ stat.label }}</p>
      </div>
    </div>

    <div class="cluster-info" v-if="currentCluster">
      <h2>Current Cluster</h2>
      <table>
        <tr>
          <td>Name</td>
          <td>{{ currentCluster.spec.displayName }}</td>
        </tr>
        <tr>
          <td>State</td>
          <td>{{ currentCluster.metadata.state.name }}</td>
        </tr>
        <tr>
          <td>Provider</td>
          <td>{{ currentCluster.status.provider }}</td>
        </tr>
      </table>
    </div>
  </div>
</template>

<script>
export default {
  name: 'Overview',

  async fetch() {
    this.nodes = await this.$store.dispatch('cluster/findAll', {
      type: 'node'
    });
    this.pods = await this.$store.dispatch('cluster/findAll', {
      type: 'pod'
    });
  },

  data() {
    return {
      nodes: [],
      pods: [],
    };
  },

  computed: {
    currentCluster() {
      return this.$store.getters['currentCluster'];
    },
    stats() {
      return [
        { label: 'Nodes', value: this.nodes.length },
        { label: 'Total Pods', value: this.pods.length },
        {
          label: 'Running Pods',
          value: this.pods.filter(p => p.status?.phase === 'Running').length
        },
        {
          label: 'Namespaces',
          value: new Set(this.pods.map(p => p.metadata?.namespace)).size
        }
      ];
    }
  }
};
</script>

<style scoped>
.overview-page {
  padding: 20px;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
  margin: 20px 0;
}

.stat-card {
  background: var(--body-bg);
  border: 1px solid var(--border);
  border-radius: 4px;
  padding: 16px;
  text-align: center;
}

.stat-card h3 {
  font-size: 2em;
  margin: 0;
}
</style>
```

### Register Routes

Update `pkg/my-extension/index.ts` to include routes:

```typescript
import { importTypes } from '@rancher/auto-import';
import { IPlugin } from '@shell/core/types';

const routes = [
  {
    name: 'c-cluster-my-extension-overview',
    path: '/c/:cluster/my-extension/overview',
    component: () => import('./pages/overview.vue'),
    meta: {
      product: 'my-extension',
      cluster: true,
    }
  }
];

const extension: IPlugin = {
  name: 'my-extension',
  routes,
  stores: [],
};

export default extension;
```

## Adding Custom Resource Views

### Create a List View for a Custom Resource

Create `pkg/my-extension/list/my-resource.vue`:

```vue
<template>
  <ResourceTable
    :schema="schema"
    :rows="rows"
    :headers="headers"
  />
</template>

<script>
import ResourceTable from '@shell/components/ResourceTable';

export default {
  name: 'MyResourceList',
  components: { ResourceTable },

  props: {
    resource: {
      type: String,
      required: true,
    }
  },

  async fetch() {
    this.rows = await this.$store.dispatch('cluster/findAll', {
      type: this.resource,
    });
    this.schema = this.$store.getters['cluster/schemaFor'](this.resource);
  },

  data() {
    return {
      rows: [],
      schema: null,
    };
  },

  computed: {
    headers() {
      return [
        { name: 'name', label: 'Name', value: 'metadata.name' },
        { name: 'namespace', label: 'Namespace', value: 'metadata.namespace' },
        { name: 'age', label: 'Age', value: 'metadata.creationTimestamp' },
      ];
    }
  }
};
</script>
```

## Adding Actions and Buttons

### Custom Action on Resources

```typescript
// pkg/my-extension/config/table-actions.ts
export function init($plugin: IPlugin, store: any) {
  const { configureType } = $plugin.DSL(store, $plugin.name);

  configureType('apps.deployment', {
    customActions: [
      {
        action: 'restartDeployment',
        label: 'Restart',
        icon: 'icon-refresh',
        enabled: (resource: any) => resource.metadata?.state?.name === 'active',
      }
    ]
  });
}
```

## Building for Production

### Build the Extension

```bash
yarn build-pkg my-extension
```

This creates a production bundle in the `dist-pkg` directory.

### Create a Helm Chart for Distribution

```bash
yarn build-helm my-extension
```

The Helm chart is generated in `charts/my-extension/`.

### Publish to a Helm Repository

```bash
# Package the chart
helm package charts/my-extension -d dist-charts/

# Push to your chart repository
helm push dist-charts/my-extension-*.tgz oci://registry.example.com/charts
```

## Deploying the Extension

### Deploy via Helm

```bash
helm install my-extension oci://registry.example.com/charts/my-extension \
  --namespace cattle-ui-plugin-system \
  --create-namespace
```

### Deploy via the Rancher UI

1. Go to **Extensions** in the Rancher sidebar
2. Click **Install from Helm Repository**
3. Add your Helm chart repository URL
4. Select and install your extension

## Testing Your Extension

### Unit Tests

```bash
yarn test
```

### End-to-End Testing

Use Cypress for E2E testing:

```bash
yarn test:e2e
```

## Summary

Building Rancher UI Extensions involves scaffolding a project with the Rancher extension creator, defining products and routes, creating Vue components for custom pages, and packaging everything as a Helm chart for distribution. The extension system provides access to the Rancher store for data fetching, built-in UI components like ResourceTable, and a DSL for registering navigation items and resource configurations.
