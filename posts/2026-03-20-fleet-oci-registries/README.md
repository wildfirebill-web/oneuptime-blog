# How to Set Up Fleet with OCI Registries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Rancher, Kubernetes, OCI

Description: Learn how to configure Fleet to deploy applications from OCI (Open Container Initiative) registries, enabling GitOps workflows based on container registries instead of Git.

## Introduction

Fleet supports OCI (Open Container Initiative) registries as a source for bundles, in addition to Git repositories. This enables you to package your Kubernetes configurations as OCI artifacts and push them to container registries like Docker Hub, Amazon ECR, Google Artifact Registry, or Harbor. When Fleet detects a new artifact version in the registry, it automatically deploys the updated configuration to your clusters.

## Prerequisites

- Fleet installed in Rancher (v0.9+ for OCI support)
- An OCI-compatible container registry
- `kubectl` access to Fleet manager
- `helm` and `oras` CLI tools

## Understanding OCI Bundles in Fleet

OCI bundles work similarly to Git-based bundles, but instead of pointing to a Git repository URL, you point Fleet at an OCI registry URL. Fleet pulls the OCI artifact, extracts the Kubernetes manifests, and deploys them.

OCI artifacts can contain:
- Raw Kubernetes YAML files
- Helm charts
- Kustomize overlays
- fleet.yaml configurations

## Packaging Manifests as OCI Artifacts

### Using Helm to Package and Push

```bash
# Package a Helm chart as an OCI artifact

helm package ./my-chart

# Log into OCI registry (example: Docker Hub)
helm registry login registry-1.docker.io \
  --username my-username \
  --password my-token

# Push chart to OCI registry
helm push my-chart-1.0.0.tgz \
  oci://registry-1.docker.io/my-org

# Push to Amazon ECR (after aws ecr get-login-password)
helm push my-chart-1.0.0.tgz \
  oci://123456789.dkr.ecr.us-east-1.amazonaws.com/charts
```

### Using ORAS to Push Raw Manifests

```bash
# Install ORAS CLI
# https://oras.land/docs/installation

# Package directory as OCI artifact
oras push \
  registry.example.com/my-org/k8s-configs:v1.0.0 \
  --artifact-type application/vnd.fleet.bundle.v1 \
  ./k8s-manifests/:application/yaml

# Push to a private registry with authentication
oras push \
  --username myuser \
  --password mytoken \
  registry.example.com/k8s-configs:v1.0.0 \
  ./manifests/
```

## Creating a GitRepo for OCI Sources

```yaml
# gitrepo-oci.yaml - Reference an OCI registry as source
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app-oci
  namespace: fleet-default
spec:
  # OCI URL format: oci://registry/repository:tag
  repo: oci://registry.example.com/my-org/my-app

  # OCI reference (tag or digest)
  # Using a digest provides immutable pinning
  revision: v1.2.0
  # Or use a digest: sha256:abc123...

  # Authentication secret for private registry
  clientSecretName: oci-registry-auth

  targets:
    - clusterSelector: {}
```

## Setting Up OCI Registry Authentication

### For Docker Hub

```bash
# Create secret from Docker config
kubectl create secret docker-registry oci-registry-auth \
  --docker-server=registry-1.docker.io \
  --docker-username=my-username \
  --docker-password=my-access-token \
  -n fleet-default
```

### For Amazon ECR

```bash
# Get ECR login token (valid for 12 hours)
AWS_TOKEN=$(aws ecr get-login-password --region us-east-1)

# Create secret
kubectl create secret docker-registry ecr-registry-auth \
  --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="${AWS_TOKEN}" \
  -n fleet-default
```

### For Harbor

```bash
# Create secret for Harbor registry
kubectl create secret docker-registry harbor-registry-auth \
  --docker-server=harbor.example.com \
  --docker-username=robot\$my-robot-account \
  --docker-password=my-robot-token \
  -n fleet-default
```

## GitRepo Using OCI with Helm Charts

```yaml
# gitrepo-oci-helm.yaml - Deploy Helm chart from OCI registry
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-helm-app-oci
  namespace: fleet-default
spec:
  # Point to OCI-hosted Helm chart
  repo: oci://registry.example.com/charts/my-app

  # Chart version to deploy
  revision: "2.1.0"

  clientSecretName: harbor-registry-auth

  targets:
    - name: production
      clusterSelector:
        matchLabels:
          env: production
      helm:
        values:
          replicaCount: 3
```

## Automating OCI Updates in CI/CD

Set up a CI/CD pipeline to push OCI artifacts and update Fleet:

```yaml
# .github/workflows/deploy-oci.yml
name: Package and Deploy via OCI

on:
  push:
    branches: [main]
    paths: ['k8s/**']

jobs:
  package-and-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install ORAS
        run: |
          ORAS_VERSION=1.1.0
          curl -LO https://github.com/oras-project/oras/releases/download/v${ORAS_VERSION}/oras_${ORAS_VERSION}_linux_amd64.tar.gz
          tar -xf oras_${ORAS_VERSION}_linux_amd64.tar.gz oras
          sudo mv oras /usr/local/bin/

      - name: Push manifests to OCI registry
        run: |
          oras push \
            ${{ secrets.REGISTRY }}/k8s-configs:${{ github.sha }} \
            ${{ secrets.REGISTRY }}/k8s-configs:latest \
            ./k8s/

      - name: Update Fleet GitRepo revision
        run: |
          # Update the GitRepo to use the new digest
          kubectl patch gitrepo my-app-oci \
            -n fleet-default \
            --type=merge \
            -p "{\"spec\":{\"revision\":\"${{ github.sha }}\"}}"
```

## Monitoring OCI-Based Deployments

```bash
# Check GitRepo status for OCI source
kubectl get gitrepo my-app-oci -n fleet-default -o wide

# View the current OCI reference being used
kubectl get gitrepo my-app-oci -n fleet-default \
  -o jsonpath='{.spec.revision}'

# Check bundle deployment status
kubectl get bundles -n fleet-default | grep my-app-oci
```

## Conclusion

OCI registry support in Fleet opens up new possibilities for packaging and distributing Kubernetes configurations. By treating your manifests as versioned OCI artifacts, you get immutable, content-addressed deployments with the same ecosystem of tools used for container images. For organizations that are already deeply integrated with container registries and CI/CD pipelines that build and push images, OCI-based Fleet deployments provide a natural extension of those workflows to Kubernetes configuration management.
