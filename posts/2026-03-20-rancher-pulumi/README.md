# How to Use Pulumi with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Pulumi, Infrastructure as Code, TypeScript

Description: Manage Rancher resources using Pulumi's infrastructure-as-code platform with real programming languages like TypeScript and Python for more expressive infrastructure definitions.

## Introduction

Pulumi allows you to define infrastructure using real programming languages (TypeScript, Python, Go, C#), providing the full power of loops, conditions, and functions for infrastructure code. The Rancher2 Pulumi provider covers the same resources as the Terraform provider but with a more developer-friendly experience. This guide covers using Pulumi with TypeScript to manage Rancher.

## Prerequisites

- Pulumi CLI installed (`npm install -g pulumi`)
- Node.js 18+ (for TypeScript)
- Rancher API token
- npm or yarn

## Step 1: Set Up a Pulumi Project

```bash
# Create a new Pulumi project
mkdir rancher-infrastructure && cd rancher-infrastructure
pulumi new typescript

# Install the Rancher2 provider
npm install @pulumi/rancher2 @pulumi/kubernetes

# Configure Rancher authentication
pulumi config set rancher2:apiUrl https://rancher.example.com
pulumi config set --secret rancher2:tokenKey "token-xxxxx:yyyyyyyyyy"
```

## Step 2: Create Clusters with TypeScript

```typescript
// index.ts - Rancher cluster management with Pulumi
import * as rancher2 from "@pulumi/rancher2";
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();
const environment = config.get("environment") || "production";

// Create cloud credential for AWS
const awsCredential = new rancher2.CloudCredential("aws-credential", {
  name: `aws-${environment}`,
  amazonec2CredentialConfig: {
    accessKey: config.requireSecret("awsAccessKey"),
    secretKey: config.requireSecret("awsSecretKey"),
  },
});

// Create machine config for worker nodes
const workerMachineConfig = new rancher2.MachineConfigV2("worker-config", {
  generateName: `worker-${environment}-`,
  amazonec2Config: {
    ami: "ami-0c02fb55956c7d316",
    region: "us-east-1",
    securityGroups: ["rancher-nodes"],
    subnetId: config.require("subnetId"),
    vpcId: config.require("vpcId"),
    zone: "a",
    instanceType: "t3.xlarge",
    rootSize: "100",
  },
});

// Create the cluster
const cluster = new rancher2.ClusterV2(`cluster-${environment}`, {
  name: `production-${environment}`,
  kubernetesVersion: "v1.29.4+rke2r1",
  rkeConfig: {
    machinePools: [
      {
        name: "control-plane",
        cloudCredentialSecretName: awsCredential.name,
        controlPlaneRole: true,
        etcdRole: true,
        quantity: 3,
        machineConfig: {
          kind: workerMachineConfig.kind,
          name: workerMachineConfig.name,
        },
      },
      {
        name: "workers",
        cloudCredentialSecretName: awsCredential.name,
        workerRole: true,
        quantity: environment === "production" ? 5 : 2,
        machineConfig: {
          kind: workerMachineConfig.kind,
          name: workerMachineConfig.name,
        },
      },
    ],
  },
  labels: {
    environment: environment,
    "managed-by": "pulumi",
  },
});

// Export cluster information
export const clusterId = cluster.clusterV1Id;
export const kubeconfig = cluster.kubeConfig;
```

## Step 3: Manage Projects and RBAC

```typescript
// projects.ts - Project and RBAC management
import * as rancher2 from "@pulumi/rancher2";

interface ProjectConfig {
  name: string;
  clusterId: pulumi.Input<string>;
  cpuLimit: string;
  memoryLimit: string;
}

function createProject(config: ProjectConfig): rancher2.Project {
  return new rancher2.Project(`project-${config.name}`, {
    name: config.name,
    clusterId: config.clusterId,
    resourceQuota: {
      projectLimit: {
        limitsCpu: config.cpuLimit,
        limitsMemory: config.memoryLimit,
      },
      namespaceDefaultLimit: {
        limitsCpu: "500m",
        limitsMemory: "512Mi",
      },
    },
    containerResourceLimit: {
      limitsCpu: "200m",
      limitsMemory: "256Mi",
      requestsCpu: "50m",
      requestsMemory: "64Mi",
    },
  });
}

// Create multiple projects
const teams = [
  { name: "frontend-team", cpuLimit: "4000m", memoryLimit: "8Gi" },
  { name: "backend-team", cpuLimit: "8000m", memoryLimit: "16Gi" },
  { name: "data-team", cpuLimit: "16000m", memoryLimit: "32Gi" },
];

const projects = teams.map(team =>
  createProject({
    name: team.name,
    clusterId: cluster.clusterV1Id,
    cpuLimit: team.cpuLimit,
    memoryLimit: team.memoryLimit,
  })
);
```

## Step 4: Deploy Helm Charts

```typescript
// apps.ts - Deploy Helm charts via Pulumi
import * as rancher2 from "@pulumi/rancher2";

// Install cert-manager
const certManager = new rancher2.AppV2("cert-manager", {
  clusterId: cluster.clusterV1Id,
  namespace: "cert-manager",
  name: "cert-manager",
  repoName: "rancher-charts",
  chartName: "cert-manager",
  chartVersion: "1.14.0",
  values: pulumi.interpolate`
    installCRDs: true
    prometheus:
      enabled: true
  `,
});

// Install Rancher Monitoring after cert-manager
const monitoring = new rancher2.AppV2("rancher-monitoring", {
  clusterId: cluster.clusterV1Id,
  namespace: "cattle-monitoring-system",
  name: "rancher-monitoring",
  repoName: "rancher-charts",
  chartName: "rancher-monitoring",
  chartVersion: "103.0.0",
  values: `
    prometheus:
      prometheusSpec:
        retention: 30d
  `,
}, { dependsOn: [certManager] });
```

## Step 5: Dynamic Multi-Environment Configuration

```typescript
// multi-env.ts - Environment-specific configurations
import * as pulumi from "@pulumi/pulumi";
import * as rancher2 from "@pulumi/rancher2";

const stack = pulumi.getStack();

// Environment-specific settings
const envConfigs: Record<string, { nodeCount: number; instanceType: string; retention: string }> = {
  development: { nodeCount: 1, instanceType: "t3.medium", retention: "7d" },
  staging: { nodeCount: 2, instanceType: "t3.large", retention: "14d" },
  production: { nodeCount: 5, instanceType: "t3.xlarge", retention: "30d" },
};

const envConfig = envConfigs[stack] ?? envConfigs.development;

// Create cluster with environment-specific sizing
const cluster = new rancher2.ClusterV2(`cluster-${stack}`, {
  name: `cluster-${stack}`,
  kubernetesVersion: "v1.29.4+rke2r1",
  // ... cluster config
});

// Deploy monitoring with environment-specific retention
const monitoring = new rancher2.AppV2(`monitoring-${stack}`, {
  clusterId: cluster.clusterV1Id,
  values: JSON.stringify({
    prometheus: {
      prometheusSpec: {
        retention: envConfig.retention,
      },
    },
  }),
});
```

## Step 6: State Management and CI/CD

```bash
# Use Pulumi Cloud for state management
pulumi login  # Uses Pulumi Cloud by default

# Or use S3 for state
pulumi login s3://my-pulumi-state-bucket

# Deploy to different environments
pulumi stack select production
pulumi up

pulumi stack select staging
pulumi up

# Preview changes
pulumi preview

# Refresh state from cloud
pulumi refresh
```

```yaml
# .github/workflows/pulumi.yml - GitHub Actions CI/CD
name: Pulumi Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install
      - uses: pulumi/actions@v4
        with:
          command: up
          stack-name: production
          work-dir: ./
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
          RANCHER_TOKEN_KEY: ${{ secrets.RANCHER_TOKEN_KEY }}
```

## Conclusion

Pulumi with TypeScript provides a developer-friendly alternative to Terraform's HCL for managing Rancher infrastructure. Real programming language features—loops, conditions, functions, and type safety—make complex infrastructure patterns more maintainable. The ability to define environment-specific configurations using simple objects and functions reduces duplication significantly compared to HCL. For teams already proficient in TypeScript or Python, Pulumi offers a natural progression into infrastructure as code.
