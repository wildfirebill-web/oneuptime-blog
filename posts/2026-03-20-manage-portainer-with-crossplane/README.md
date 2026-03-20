# How to Manage Portainer with Crossplane

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Crossplane, Kubernetes, Infrastructure as Code, GitOps

Description: Use Crossplane to manage Portainer environments and configurations as Kubernetes custom resources, enabling GitOps workflows for your container management platform.

## Introduction

Crossplane is a Kubernetes add-on that enables you to manage any infrastructure as Kubernetes custom resources. By building a Crossplane provider for Portainer, you can manage Portainer environments, stacks, and configurations using familiar kubectl commands and GitOps workflows. This is particularly powerful for platform engineering teams who want a single Kubernetes-native control plane.

## Prerequisites

- Kubernetes cluster with kubectl access
- Portainer installed and accessible
- Crossplane installed in your cluster
- Helm 3 installed

## Step 1: Install Crossplane

```bash
# Add Crossplane Helm repository
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

# Install Crossplane
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --wait

# Verify installation
kubectl get pods -n crossplane-system
kubectl get crds | grep crossplane
```

## Step 2: Create a Portainer Crossplane Provider

Since there isn't an official Portainer provider, we'll use Crossplane's provider-http to interact with the Portainer API:

```bash
# Install the HTTP provider for Portainer API calls
kubectl apply -f - <<'EOF'
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-http
spec:
  package: xpkg.upbound.io/upbound/provider-http:v0.3.0
EOF

# Wait for provider to be ready
kubectl wait --for=condition=Installed provider/provider-http --timeout=120s
kubectl wait --for=condition=Healthy provider/provider-http --timeout=120s
```

## Step 3: Configure Portainer Connection Credentials

```yaml
# portainer-credentials.yaml
apiVersion: v1
kind: Secret
metadata:
  name: portainer-credentials
  namespace: crossplane-system
type: Opaque
stringData:
  # Portainer API token (generate via Settings > Users > API key)
  token: "your-portainer-api-token"
  baseUrl: "https://portainer.example.com:9443/api"
---
# ProviderConfig for Portainer HTTP API
apiVersion: http.upbound.io/v1alpha1
kind: ProviderConfig
metadata:
  name: portainer-http
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: portainer-credentials
      key: token
  headers:
    Content-Type: "application/json"
    X-API-Key: "{{ .credentials }}"
```

## Step 4: Define Composite Resources for Portainer

Create XRDs (Composite Resource Definitions) to abstract Portainer concepts:

```yaml
# portainer-environment-xrd.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xportainerenvironments.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: XPortainerEnvironment
    plural: xportainerenvironments
  claimNames:
    kind: PortainerEnvironment
    plural: portainerenvironments
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                required:
                  - name
                  - dockerUrl
                properties:
                  name:
                    type: string
                    description: Name of the Portainer environment
                  dockerUrl:
                    type: string
                    description: Docker socket or TCP URL
                  environmentType:
                    type: integer
                    default: 1
                    description: "1=Docker, 2=Swarm, 6=Kubernetes"
                  tags:
                    type: array
                    items:
                      type: string
```

## Step 5: Create Composition for Portainer Environments

```yaml
# portainer-environment-composition.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: portainer-environment
  labels:
    crossplane.io/xrd: xportainerenvironments.platform.example.com
spec:
  compositeTypeRef:
    apiVersion: platform.example.com/v1alpha1
    kind: XPortainerEnvironment
  
  resources:
  - name: portainer-environment-api-call
    base:
      apiVersion: http.upbound.io/v1alpha1
      kind: Request
      spec:
        providerConfigRef:
          name: portainer-http
        forProvider:
          url: "https://portainer.example.com:9443/api/endpoints"
          method: POST
          body: ""
          headers:
            Content-Type:
              - "application/json"
    
    patches:
    # Patch the request body with environment parameters
    - type: CombineFromComposite
      combine:
        variables:
        - fromFieldPath: spec.parameters.name
        - fromFieldPath: spec.parameters.dockerUrl
        - fromFieldPath: spec.parameters.environmentType
        strategy: string
        string:
          fmt: '{"Name":"%s","URL":"%s","EndpointCreationType":%d}'
      toFieldPath: spec.forProvider.body
```

## Step 6: Create Portainer Stack Custom Resource

```yaml
# portainer-stack-xrd.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xportainerstacks.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: XPortainerStack
    plural: xportainerstacks
  claimNames:
    kind: PortainerStack
    plural: portainerstacks
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                required:
                  - name
                  - environmentId
                  - gitRepository
                properties:
                  name:
                    type: string
                  environmentId:
                    type: integer
                  gitRepository:
                    type: object
                    properties:
                      url:
                        type: string
                      branch:
                        type: string
                        default: main
                      composeFilePath:
                        type: string
                        default: docker-compose.yml
```

## Step 7: Deploy Resources Using Crossplane Claims

```yaml
# my-app-environment.yaml
apiVersion: platform.example.com/v1alpha1
kind: PortainerEnvironment
metadata:
  name: production-docker
  namespace: platform
spec:
  parameters:
    name: "Production Docker"
    dockerUrl: "tcp://prod-host:2376"
    environmentType: 1
    tags:
      - production
      - docker
---
# my-app-stack.yaml
apiVersion: platform.example.com/v1alpha1
kind: PortainerStack
metadata:
  name: web-application
  namespace: platform
spec:
  parameters:
    name: web-app-production
    environmentId: 1
    gitRepository:
      url: https://github.com/my-org/my-app
      branch: main
      composeFilePath: docker/docker-compose.prod.yml
```

Apply via kubectl:

```bash
# Deploy the environment
kubectl apply -f my-app-environment.yaml

# Deploy the stack
kubectl apply -f my-app-stack.yaml

# Check resource status
kubectl get portainerenvironments -n platform
kubectl get portainerstacks -n platform

# View detailed status
kubectl describe portainerstack web-application -n platform
```

## Step 8: GitOps Integration

Store all Crossplane resources in Git for GitOps workflows:

```bash
# Example GitOps directory structure
gitops/
├── environments/
│   ├── production/
│   │   ├── portainer-environment.yaml
│   │   └── portainer-stacks/
│   │       ├── web-app.yaml
│   │       └── monitoring.yaml
│   └── staging/
│       └── ...
└── compositions/
    ├── portainer-environment-composition.yaml
    └── portainer-stack-composition.yaml
```

Apply with FluxCD or ArgoCD:

```yaml
# flux-portainer-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: portainer-platform
  namespace: flux-system
spec:
  interval: 5m
  path: ./gitops/environments/production
  sourceRef:
    kind: GitRepository
    name: platform-repo
  prune: true
```

## Conclusion

Managing Portainer with Crossplane transforms your container management platform into Kubernetes-native resources, enabling true GitOps workflows. All Portainer configurations can be stored in Git, reviewed via pull requests, and automatically applied by tools like FluxCD or ArgoCD. This approach is ideal for platform engineering teams that want to give developers a self-service experience for requesting new Portainer environments or deployments through Kubernetes custom resources.
