# How to Develop Rancher UI Plugins

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, UI Extensions, Plugins, Dashboard

Description: Comprehensive guide to developing Rancher UI plugins, covering the plugin architecture, DSL, store access, component reuse, and advanced customization patterns.

Rancher UI plugins extend the dashboard with custom functionality, resource types, and integrations. This guide goes deep into plugin development, covering the architecture, available APIs, advanced patterns, and real-world examples.

## Plugin Architecture Overview

Rancher UI plugins are Vue.js applications that run within the Rancher dashboard shell. They are loaded dynamically and have access to:

- **The Rancher Store**: Vuex store with cluster data, user info, and configuration
- **The Plugin DSL**: Methods for registering products, types, and navigation items
- **Shell Components**: Pre-built UI components like tables, forms, and banners
- **Cluster API Proxy**: Access to Kubernetes APIs through the Rancher proxy

## Creating a Plugin Project

### Initialize the Project

```bash
npx @rancher/create-extension my-plugin
cd my-plugin
yarn install
```

### Project Structure

```plaintext
my-plugin/
  pkg/
    my-plugin/
      index.ts              # Plugin entry point
      product.ts            # Product registration
      types.ts              # Custom type definitions
      pages/                # Page components
      components/           # Reusable components
      models/               # Resource model overrides
      store/                # Custom Vuex stores
      l10n/                 # Translations
        en-us.yaml
  package.json
  tsconfig.json
```

## The Plugin DSL

The Plugin DSL provides methods for registering your plugin with the Rancher shell.

### Registering a Product

```typescript
// product.ts
import { IPlugin } from '@shell/core/types';

export function init($plugin: IPlugin, store: any) {
  const {
    product,
    basicType,
    virtualType,
    configureType,
    weightType,
    headers,
  } = $plugin.DSL(store, $plugin.name);

  // Register product in the top navigation
  product({
    icon: 'cluster',
    inStore: 'management', // or 'cluster' for cluster-scoped
    weight: 100,
    to: {
      name: `c-cluster-${$plugin.name}-overview`,
      params: { product: $plugin.name }
    }
  });

  // Register resource types
  virtualType({
    name: 'overview',
    route: {
      name: `c-cluster-${$plugin.name}-overview`,
      params: { product: $plugin.name }
    }
  });

  virtualType({
    name: 'settings',
    route: {
      name: `c-cluster-${$plugin.name}-settings`,
      params: { product: $plugin.name }
    }
  });

  // Group types in navigation
  basicType(['overview', 'settings']);
}
```

### Registering Real Kubernetes Resource Types

```typescript
export function init($plugin: IPlugin, store: any) {
  const { product, basicType, configureType, weightType } = $plugin.DSL(store, $plugin.name);

  product({
    icon: 'gear',
    inStore: 'cluster',
    weight: 90,
  });

  // Register a CRD type
  configureType('my.company.io.myresource', {
    isCreatable: true,
    isEditable: true,
    isRemovable: true,
    showAge: true,
    showState: true,
  });

  // Set display weight (order in navigation)
  weightType('my.company.io.myresource', 100);

  basicType(['my.company.io.myresource']);
}
```

## Working with the Rancher Store

### Fetching Resources

```typescript
// In a Vue component
export default {
  async fetch() {
    // Fetch all resources of a type
    this.deployments = await this.$store.dispatch('cluster/findAll', {
      type: 'apps.deployment'
    });

    // Fetch a single resource
    this.myPod = await this.$store.dispatch('cluster/find', {
      type: 'pod',
      id: 'default/my-pod'
    });

    // Fetch with custom options
    this.nodes = await this.$store.dispatch('cluster/findAll', {
      type: 'node',
      opt: { force: true }  // Force refresh from API
    });
  }
};
```

### Accessing Schemas

```typescript
// Check if a resource type exists
const schema = this.$store.getters['cluster/schemaFor']('apps.deployment');
if (schema) {
  console.log('Deployment schema available');
}

// Get all schemas
const allSchemas = this.$store.getters['cluster/all']('schema');
```

### Making Custom API Requests

```typescript
// GET request through the cluster proxy
const response = await this.$store.dispatch('cluster/request', {
  url: '/api/v1/namespaces',
  method: 'GET'
});

// POST request
await this.$store.dispatch('cluster/request', {
  url: '/api/v1/namespaces',
  method: 'POST',
  data: {
    apiVersion: 'v1',
    kind: 'Namespace',
    metadata: { name: 'new-namespace' }
  }
});
```

## Custom Resource Models

Override how resources are displayed and behave by creating model classes:

```typescript
// models/my.company.io.myresource.js
import SteveModel from '@shell/plugins/steve/steve-class';

export default class MyResource extends SteveModel {
  // Custom display name
  get nameDisplay() {
    return this.spec?.friendlyName || this.metadata?.name || 'Unknown';
  }

  // Custom state computation
  get stateDisplay() {
    if (this.status?.ready) return 'Ready';
    if (this.status?.processing) return 'Processing';
    return 'Pending';
  }

  // Custom state color
  get stateColor() {
    const state = this.stateDisplay;
    if (state === 'Ready') return 'success';
    if (state === 'Processing') return 'info';
    return 'warning';
  }

  // Available actions in the context menu
  get _availableActions() {
    const actions = super._availableActions;

    actions.unshift({
      action: 'customAction',
      label: 'Run Diagnostics',
      icon: 'icon-search',
      enabled: this.stateDisplay === 'Ready',
    });

    return actions;
  }

  // Implement the custom action
  customAction() {
    this.$dispatch('cluster/request', {
      url: `/apis/my.company.io/v1/namespaces/${this.metadata.namespace}/myresources/${this.metadata.name}/diagnostics`,
      method: 'POST'
    });
  }

  // Custom detail page tabs
  get detailTabs() {
    return [
      { name: 'overview', label: 'Overview', weight: 100 },
      { name: 'config', label: 'Configuration', weight: 90 },
      { name: 'logs', label: 'Logs', weight: 80 },
    ];
  }
}
```

## Custom List Columns

Define custom columns for resource list views:

```typescript
// product.ts
import { IPlugin } from '@shell/core/types';

export function init($plugin: IPlugin, store: any) {
  const { headers, configureType } = $plugin.DSL(store, $plugin.name);

  headers('my.company.io.myresource', [
    {
      name: 'name',
      labelKey: 'Name',
      value: 'metadata.name',
      sort: ['metadata.name'],
      width: 200,
    },
    {
      name: 'status',
      labelKey: 'Status',
      value: 'stateDisplay',
      sort: ['stateDisplay'],
      width: 120,
      formatter: 'BadgeState',
    },
    {
      name: 'version',
      labelKey: 'Version',
      value: 'spec.version',
      sort: ['spec.version'],
    },
    {
      name: 'replicas',
      labelKey: 'Replicas',
      value: 'spec.replicas',
      sort: ['spec.replicas:desc'],
      width: 100,
    },
    {
      name: 'age',
      labelKey: 'Age',
      value: 'metadata.creationTimestamp',
      sort: ['metadata.creationTimestamp'],
      formatter: 'LiveDate',
      width: 120,
    }
  ]);
}
```

## Custom Create and Edit Forms

Create forms for your custom resources:

```vue
<!-- edit/my.company.io.myresource.vue -->
<template>
  <CruResource
    :resource="value"
    :mode="mode"
    :errors="errors"
    @finish="save"
    @error="e => errors = e"
  >
    <div class="row mb-20">
      <div class="col span-6">
        <LabeledInput
          v-model="value.metadata.name"
          label="Name"
          :mode="mode"
          required
        />
      </div>
      <div class="col span-6">
        <LabeledInput
          v-model="value.spec.version"
          label="Version"
          :mode="mode"
        />
      </div>
    </div>

    <div class="row mb-20">
      <div class="col span-6">
        <LabeledInput
          v-model.number="value.spec.replicas"
          label="Replicas"
          type="number"
          :mode="mode"
          :min="1"
          :max="100"
        />
      </div>
      <div class="col span-6">
        <LabeledSelect
          v-model="value.spec.tier"
          label="Tier"
          :options="tierOptions"
          :mode="mode"
        />
      </div>
    </div>

    <div class="row mb-20">
      <div class="col span-12">
        <KeyValue
          v-model="value.spec.config"
          title="Configuration"
          :mode="mode"
          :add-label="'Add Config Entry'"
        />
      </div>
    </div>
  </CruResource>
</template>

<script>
import CruResource from '@shell/components/CruResource';
import LabeledInput from '@components/Form/LabeledInput/LabeledInput';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import KeyValue from '@shell/components/form/KeyValue';
import CreateEditView from '@shell/mixins/create-edit-view';

export default {
  name: 'MyResourceEdit',
  components: {
    CruResource,
    LabeledInput,
    LabeledSelect,
    KeyValue,
  },
  mixins: [CreateEditView],

  data() {
    return {
      tierOptions: [
        { label: 'Free', value: 'free' },
        { label: 'Standard', value: 'standard' },
        { label: 'Premium', value: 'premium' },
      ],
    };
  },
};
</script>
```

## Internationalization (i18n)

Add translations for your plugin:

```yaml
# l10n/en-us.yaml

product:
  my-plugin: "My Plugin"

typeLabel:
  "my.company.io.myresource": |-
    {count, plural,
      one { My Resource }
      other { My Resources }
    }

nav:
  overview: "Overview"
  settings: "Settings"
```

Load translations in your plugin entry:

```typescript
// index.ts
import { importTypes } from '@rancher/auto-import';
import { IPlugin } from '@shell/core/types';

const extension: IPlugin = {
  name: 'my-plugin',
  routes: [...],
  stores: [],
};

export default extension;
```

## Development Workflow

### Running in Development Mode

```bash
# Set the Rancher API URL
API=https://rancher.example.com yarn dev
```

### Hot Module Replacement

The development server supports HMR, so changes to Vue components are reflected immediately without a full page reload.

### Debugging

Use Vue DevTools in your browser to inspect the Vuex store, component hierarchy, and events. The Rancher store contains all fetched resources under `cluster/all` getters.

### Building and Publishing

```bash
# Build the plugin package
yarn build-pkg my-plugin

# Build a Helm chart
yarn build-helm my-plugin

# Package and publish
helm package charts/my-plugin
helm push my-plugin-1.0.0.tgz oci://registry.example.com/charts
```

## Summary

Developing Rancher UI plugins involves using the Plugin DSL to register products and resource types, creating Vue components for pages and forms, defining custom resource models for behavior overrides, and packaging everything as a Helm chart. The plugin architecture provides full access to the Rancher store for data fetching, pre-built shell components for consistent UI, and internationalization support. Start with simple virtual types and pages, then progress to custom CRD management with models, list columns, and edit forms.
