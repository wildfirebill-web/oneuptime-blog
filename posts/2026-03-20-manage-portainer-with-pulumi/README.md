# How to Manage Portainer with Pulumi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Pulumi, Infrastructure as Code, DevOps, Automation

Description: Use Pulumi's infrastructure as code platform to automate Portainer deployments, environment management, and stack configurations using TypeScript or Python.

## Introduction

Pulumi is a modern infrastructure as code platform that lets you use familiar programming languages like TypeScript, Python, and Go to define cloud infrastructure. Unlike HCL-based tools, Pulumi gives you the full power of a programming language with loops, conditionals, and abstractions. By combining Pulumi with Portainer's REST API, you can fully automate your container management infrastructure.

## Prerequisites

- Pulumi CLI installed: `curl -fsSL https://get.pulumi.com | sh`
- Node.js 16+ (for TypeScript) or Python 3.9+
- Portainer instance running and accessible
- Portainer API access token

## Step 1: Initialize Pulumi Project

```bash
# Create new Pulumi project

mkdir portainer-pulumi && cd portainer-pulumi
pulumi new typescript

# Or for Python
pulumi new python

# Install dependencies
npm install @pulumi/pulumi @pulumi/docker axios

# Set Portainer configuration
pulumi config set portainerUrl https://portainer.example.com:9443
pulumi config set --secret portainerPassword your-secure-password
```

## Step 2: Create Portainer Provider Helper

Since Pulumi doesn't have a native Portainer provider, we'll create a custom resource provider:

```typescript
// portainerProvider.ts
import * as pulumi from "@pulumi/pulumi";
import axios, { AxiosInstance } from "axios";

export class PortainerClient {
  private client: AxiosInstance;
  private token: string = "";
  
  constructor(
    private baseUrl: string,
    private username: string,
    private password: string
  ) {
    this.client = axios.create({
      baseURL: `${baseUrl}/api`,
      // Skip TLS verification for self-signed certs in dev
      httpsAgent: new (require("https").Agent)({ rejectUnauthorized: false }),
    });
  }

  async authenticate(): Promise<void> {
    const response = await this.client.post("/auth", {
      Username: this.username,
      Password: this.password,
    });
    this.token = response.data.jwt;
    this.client.defaults.headers.common["Authorization"] = `Bearer ${this.token}`;
  }

  async getEnvironments(): Promise<any[]> {
    const response = await this.client.get("/endpoints");
    return response.data;
  }

  async createStack(
    envId: number,
    name: string,
    composeContent: string,
    envVars: Record<string, string> = {}
  ): Promise<any> {
    const env = Object.entries(envVars).map(([name, value]) => ({
      name,
      value,
    }));
    
    const response = await this.client.post(
      `/stacks/create/standalone/string?endpointId=${envId}`,
      {
        Name: name,
        StackFileContent: composeContent,
        Env: env,
      }
    );
    return response.data;
  }

  async deleteStack(stackId: number): Promise<void> {
    await this.client.delete(`/stacks/${stackId}`);
  }
}
```

## Step 3: Define Portainer Stack as Pulumi Resource

```typescript
// portainerStack.ts
import * as pulumi from "@pulumi/pulumi";
import { PortainerClient } from "./portainerProvider";

interface PortainerStackArgs {
  name: pulumi.Input<string>;
  environmentId: pulumi.Input<number>;
  composeContent: pulumi.Input<string>;
  envVars?: pulumi.Input<Record<string, string>>;
}

export class PortainerStack extends pulumi.dynamic.Resource {
  public readonly stackId!: pulumi.Output<number>;
  public readonly stackName!: pulumi.Output<string>;

  constructor(
    name: string,
    args: PortainerStackArgs,
    opts?: pulumi.CustomResourceOptions
  ) {
    super(
      {
        // Create the stack in Portainer
        async create(inputs: any): Promise<pulumi.dynamic.CreateResult> {
          const config = new pulumi.Config();
          const client = new PortainerClient(
            config.require("portainerUrl"),
            "admin",
            config.requireSecret("portainerPassword")
          );
          
          await client.authenticate();
          
          const result = await client.createStack(
            inputs.environmentId,
            inputs.name,
            inputs.composeContent,
            inputs.envVars || {}
          );
          
          return {
            id: String(result.Id),
            outs: {
              stackId: result.Id,
              stackName: result.Name,
              ...inputs,
            },
          };
        },
        
        // Delete the stack from Portainer
        async delete(id: pulumi.ID, inputs: any): Promise<void> {
          const config = new pulumi.Config();
          const client = new PortainerClient(
            config.require("portainerUrl"),
            "admin",
            config.requireSecret("portainerPassword")
          );
          
          await client.authenticate();
          await client.deleteStack(parseInt(id));
        },
      },
      name,
      {
        stackId: undefined,
        stackName: undefined,
        ...args,
      },
      opts
    );
  }
}
```

## Step 4: Create the Main Pulumi Program

```typescript
// index.ts
import * as pulumi from "@pulumi/pulumi";
import * as docker from "@pulumi/docker";
import { PortainerStack } from "./portainerStack";
import * as fs from "fs";

const config = new pulumi.Config();
const environment = pulumi.getStack(); // dev, staging, prod

// Deploy monitoring stack
const monitoringComposeContent = `
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:latest
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
  
  grafana:
    image: grafana/grafana:latest
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=\${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false

volumes:
  prometheus-data:
  grafana-data:
`;

const monitoringStack = new PortainerStack("monitoring", {
  name: `monitoring-${environment}`,
  environmentId: 1, // Portainer environment ID
  composeContent: monitoringComposeContent,
  envVars: {
    GRAFANA_PASSWORD: config.requireSecret("grafanaPassword"),
  },
});

// Deploy application stack from file
const appComposeContent = fs.readFileSync(
  `./stacks/${environment}/docker-compose.yml`,
  "utf-8"
);

const appStack = new PortainerStack("web-app", {
  name: `web-app-${environment}`,
  environmentId: 1,
  composeContent: appComposeContent,
  envVars: {
    DB_PASSWORD: config.requireSecret("dbPassword"),
    APP_SECRET: config.requireSecret("appSecret"),
    ENVIRONMENT: environment,
  },
});

// Export stack information
export const monitoringStackId = monitoringStack.stackId;
export const appStackId = appStack.stackId;
export const deploymentEnvironment = environment;
```

## Step 5: Deploy with Pulumi

```bash
# Initialize Pulumi state (local backend)
pulumi login --local
# Or use Pulumi Cloud
pulumi login

# Set secrets
pulumi config set --secret grafanaPassword your-grafana-password
pulumi config set --secret dbPassword your-db-password

# Preview changes
pulumi preview

# Deploy to development
pulumi stack select dev
pulumi up --yes

# Deploy to production
pulumi stack select prod
pulumi up --yes

# View deployed resources
pulumi stack output

# Destroy resources
pulumi destroy --yes
```

## Step 6: Multi-Environment Management

```typescript
// environments.ts
import * as pulumi from "@pulumi/pulumi";

interface EnvironmentConfig {
  portainerEnvId: number;
  replicas: number;
  resources: {
    cpuLimit: string;
    memoryLimit: string;
  };
}

const environmentConfigs: Record<string, EnvironmentConfig> = {
  dev: {
    portainerEnvId: 1,
    replicas: 1,
    resources: { cpuLimit: "0.5", memoryLimit: "512m" },
  },
  staging: {
    portainerEnvId: 2,
    replicas: 2,
    resources: { cpuLimit: "1", memoryLimit: "1g" },
  },
  prod: {
    portainerEnvId: 3,
    replicas: 3,
    resources: { cpuLimit: "2", memoryLimit: "2g" },
  },
};

export function getEnvironmentConfig(
  env: string
): EnvironmentConfig {
  const config = environmentConfigs[env];
  if (!config) {
    throw new Error(`Unknown environment: ${env}`);
  }
  return config;
}
```

## Conclusion

Managing Portainer with Pulumi brings the full power of a programming language to your infrastructure automation. The ability to use TypeScript's type system, conditionals, loops, and modules makes complex deployment scenarios manageable. With Pulumi's state management, you get a reliable record of what's deployed and what changes will be applied. This approach is particularly valuable for teams that prefer code over configuration files and want to apply software engineering best practices to their infrastructure.
