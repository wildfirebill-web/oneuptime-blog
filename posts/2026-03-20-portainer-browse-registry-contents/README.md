# How to Browse Registry Contents in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Registry, Business Edition, DevOps

Description: Learn how to use Portainer Business Edition's registry browser to navigate and manage container images directly from the Portainer UI.

## Introduction

Portainer Business Edition includes a built-in registry browser that lets you explore registry contents — repositories, images, and tags — without needing separate tools. Instead of knowing image paths by memory or using the registry's own web UI, you can browse and select images directly within Portainer. This guide covers using the registry browser.

## Prerequisites

- Portainer Business Edition (BE) installed
- At least one registry configured in Portainer (Docker Hub, ECR, ACR, Harbor, etc.)
- Registry credentials with list/read permissions

## Supported Registries for Browsing

| Registry | Browsable in BE |
|---------|---------------|
| Docker Hub | Yes (authenticated) |
| Harbor | Yes |
| Custom Docker Registry v2 | Yes |
| AWS ECR | Yes |
| Azure ACR | Yes |
| GitHub GHCR | Yes |
| GitLab Registry | Yes |

## Step 1: Open the Registry Browser

1. Log in to Portainer BE
2. Click **Registries** in the left sidebar
3. Find the registry you want to browse
4. Click the **Browse** button (folder icon) next to the registry

The registry browser opens.

## Step 2: Navigate the Registry Structure

The browser shows the registry hierarchy:

```
Registry: harbor.company.com
├── production/
│   ├── myapp          (repository)
│   │   ├── latest     (tag)
│   │   ├── v2.1.0     (tag)
│   │   └── v2.0.0     (tag)
│   └── api
│       ├── latest
│       └── v3.0.1
└── staging/
    ├── myapp
    │   └── develop
    └── api
        └── develop
```

## Step 3: Explore Image Tags

1. Click on a repository to see available tags
2. Each tag shows:
   - Tag name
   - Image digest
   - Creation date
   - Image size

```
Repository: harbor.company.com/production/myapp
────────────────────────────────────────────────
Tag        Digest              Created        Size
latest     sha256:abc123...    2024-01-15     125MB
v2.1.0     sha256:def456...    2024-01-10     124MB
v2.0.0     sha256:ghi789...    2023-12-01     118MB
v1.9.5     sha256:jkl012...    2023-11-15     115MB
```

## Step 4: Use Images from the Browser

When browsing, you can click an image/tag to use it directly in a deployment:

1. Find the image and tag you want
2. Click the image row or use the **Deploy** button
3. Portainer pre-fills the image field with the full image path

This eliminates typos and ensures you use the exact image you intended.

## Step 5: Delete Images and Tags

In the registry browser (BE feature with appropriate permissions):

1. Navigate to a tag
2. Click the **Delete** icon (trash)
3. Confirm deletion

This calls the registry's delete API to remove the image manifest and free storage space.

**Warning:** Deleting a tag is irreversible. Ensure you no longer need the image before deleting.

## Step 6: Tag an Image from the Browser

From the browser, you can apply new tags to existing images:

1. Navigate to the image you want to retag
2. Click the **Tag** option
3. Enter the new tag name (e.g., `stable`, `production`)
4. Confirm

This creates a new tag pointing to the same image digest without duplicating storage.

## Step 7: Inspect Image Layers and Metadata

Click on a specific image tag to see detailed metadata:

```
Image: myapp:v2.1.0
──────────────────────────────────────────────────
Digest:       sha256:abc123def456...
Architecture: linux/amd64
OS:           linux
Created:      2024-01-15T10:00:00Z
Size:         125.3 MB

Layers:
  sha256:layer1...   45.2 MB
  sha256:layer2...   32.1 MB
  sha256:layer3...   28.7 MB
  sha256:layer4...   19.3 MB

Labels:
  org.opencontainers.image.source=https://github.com/myorg/myapp
  org.opencontainers.image.version=v2.1.0
```

## Step 8: Set Registry Access Policies (BE)

In Portainer BE, control which teams can access registry browsing:

1. Go to **Registries → {registry} → Registry access**
2. Assign access to specific teams:
   - **Admin** — Full access including delete
   - **Read only** — Can browse and pull
   - **No access** — Cannot use the registry

## Comparing Registry Browser vs Registry UI

| Feature | Portainer BE Browser | Harbor UI | AWS ECR Console |
|---------|---------------------|-----------|----------------|
| View tags | Yes | Yes | Yes |
| Select for deployment | Yes | No | No |
| Delete images | Yes | Yes | Yes |
| Scan results | No | Yes | Yes |
| Replication policies | No | Yes | No |

Use the Portainer browser for deployment workflows; use the native registry UI for advanced management.

## Conclusion

The registry browser in Portainer Business Edition streamlines the deployment workflow by letting you navigate your container image catalog without leaving the Portainer interface. Instead of memorizing image paths and tags, you browse visually and click to deploy. For teams managing multiple registries with many images, this feature significantly reduces friction in the deployment process.
