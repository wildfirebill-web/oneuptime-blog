# How to Package and Publish Rancher Extensions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Extensions, UI

Description: A step-by-step guide to packaging and publishing custom Rancher UI extensions to an OCI registry or Helm chart repository.

## Introduction

Rancher Extensions allow teams to extend the Rancher UI with custom dashboards, widgets, and workflows. Once you've built an extension, the next step is packaging it into a distributable artifact and publishing it so other Rancher instances can install it. This guide walks through the full packaging and publishing workflow.

## Prerequisites

- Node.js 16+ and Yarn installed
- A working Rancher Extension project (scaffold with `@rancher/shell`)
- Access to an OCI-compatible registry (e.g., GitHub Container Registry, Docker Hub) or a Helm repository
- `helm` CLI v3.8+ installed

## Understanding Rancher Extension Artifacts

A Rancher Extension is distributed as a Helm chart that bundles your compiled UI assets. When Rancher installs the extension chart, it serves the static assets from within the cluster and loads them dynamically into the Rancher dashboard.

## Step 1: Build the Extension

Navigate to your extension project and compile the production assets.

```bash
# Install dependencies

yarn install

# Build the extension for production
yarn build-pkg <your-extension-name>

# The compiled output lands in dist-pkg/<your-extension-name>/
ls dist-pkg/
```

## Step 2: Package into a Helm Chart

Rancher provides a helper script to wrap your built assets into a Helm chart.

```bash
# Run the packaging script bundled with @rancher/shell
yarn publish-pkgs -s <your-extension-name> -r <registry-hostname> -o <registry-org>

# Example targeting GitHub Container Registry
yarn publish-pkgs -s my-extension -r ghcr.io -o my-org
```

This script:
1. Copies compiled assets into a generated Helm chart skeleton.
2. Tags the chart with the version defined in your `package.json`.
3. Outputs the chart to `tmp/` ready for pushing.

## Step 3: Authenticate to Your Registry

```bash
# Authenticate to GitHub Container Registry
echo $GITHUB_PAT | helm registry login ghcr.io -u <github-username> --password-stdin

# Authenticate to Docker Hub OCI endpoint
echo $DOCKERHUB_TOKEN | helm registry login registry-1.docker.io -u <dockerhub-username> --password-stdin
```

## Step 4: Push the Chart to an OCI Registry

```bash
# Push the packaged chart (replace version as appropriate)
helm push tmp/my-extension-1.0.0.tgz oci://ghcr.io/my-org/charts

# Verify the artifact was pushed
helm show chart oci://ghcr.io/my-org/charts/my-extension --version 1.0.0
```

## Step 5: Create a Helm Repository Index (Optional)

If you prefer a classic HTTP Helm repository instead of OCI:

```bash
# Move the packaged chart to your repo directory
cp tmp/my-extension-1.0.0.tgz helm-repo/

# Generate or update the index
helm repo index helm-repo/ --url https://my-org.github.io/my-helm-repo

# Commit and push to GitHub Pages (or any static hosting)
git -C helm-repo add . && git -C helm-repo commit -m "Add my-extension 1.0.0" && git -C helm-repo push
```

## Step 6: Install the Extension in Rancher

1. Log in to Rancher.
2. Navigate to **Extensions** → **Add Extension Catalog**.
3. Enter the chart repository URL (`https://my-org.github.io/my-helm-repo`) or the OCI URL (`oci://ghcr.io/my-org/charts`).
4. Click **Install** next to your extension.

## Automating with GitHub Actions

```yaml
# .github/workflows/publish-extension.yaml
name: Publish Rancher Extension

on:
  push:
    tags: ["v*"]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "18"

      - run: yarn install

      - run: yarn build-pkg my-extension

      - name: Log in to GHCR
        run: echo "${{ secrets.GITHUB_TOKEN }}" | helm registry login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Package and push
        run: yarn publish-pkgs -s my-extension -r ghcr.io -o ${{ github.repository_owner }}
```

## Versioning Best Practices

- Follow **Semantic Versioning** (`MAJOR.MINOR.PATCH`).
- Bump the version in `package.json` before every release.
- Tag commits (`git tag v1.2.0`) to trigger CI/CD pipelines.

## Conclusion

Packaging and publishing Rancher Extensions involves building your UI assets, wrapping them in a Helm chart, and pushing to a registry. Once published, any Rancher instance can discover and install your extension through the built-in Extensions catalog. Automating this process with GitHub Actions ensures consistent, reproducible releases with minimal manual intervention.
