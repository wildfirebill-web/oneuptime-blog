# How to Browse Registry Contents in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Container Registry, Portainer Business Edition, Image Management, DevOps

Description: Learn how to use Portainer Business Edition's registry browser to view repositories, tags, and image details without using the CLI.

## Overview

Portainer Business Edition includes a built-in registry browser that lets you explore the contents of connected registries directly from the UI. This eliminates the need to use the Docker CLI or registry API manually to find available images and tags.

## Accessing the Registry Browser

1. Log in to Portainer Business Edition.
2. Go to **Settings > Registries**.
3. Click the registry you want to browse.
4. Click the **Browse** button (or the eye icon) next to the registry.

You will see a list of all repositories in the registry.

## What You Can Do in the Registry Browser

- **View repositories**: See all image repositories in the registry.
- **Explore tags**: Click a repository to see all available tags.
- **View image details**: See image digest, size, creation date, and layers.
- **Delete tags**: Remove old or unused tags directly from the UI.
- **Retag images**: Copy an image tag to a new name within the same registry.

## Equivalent CLI Commands

The registry browser replaces these manual CLI workflows:

```bash
# List all repositories in a registry (Docker Registry API v2)
curl -u user:password \
  https://registry.mycompany.com/v2/_catalog

# List tags for a specific repository
curl -u user:password \
  https://registry.mycompany.com/v2/myapp/tags/list

# Get image manifest details
curl -u user:password \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  https://registry.mycompany.com/v2/myapp/manifests/latest
```

## Deleting Old Tags via the Browser

Old image tags consume storage. Use the registry browser to identify and remove stale images:

1. Navigate to the repository in the browser.
2. Check the boxes next to tags you want to remove.
3. Click **Delete** and confirm.

## Note on Tag Deletion

Deleting a tag removes the reference, but the underlying layers remain until garbage collection runs:

```bash
# Run garbage collection on a self-hosted registry to reclaim storage
docker exec registry bin/registry garbage-collect \
  /etc/docker/registry/config.yml
```

## Conclusion

The registry browser in Portainer Business Edition significantly reduces the friction of managing image lifecycles. Instead of juggling API calls or CLI commands, you get a visual interface to audit, clean up, and inspect your container images.
