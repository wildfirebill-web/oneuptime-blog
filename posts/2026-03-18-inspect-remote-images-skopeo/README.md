# How to Inspect Remote Images with Skopeo

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Skopeo, Registry, Image Inspection

Description: Learn how to use Skopeo to inspect container images in remote registries without downloading them to your local machine.

---

> Skopeo inspect lets you examine image metadata, labels, layers, and configuration directly from the registry - no download required.

Before pulling a container image, you often need to know its details: what platform it targets, when it was created, what environment variables are set, or how large it is. Skopeo allows you to inspect images stored in remote registries without ever downloading them. This saves bandwidth and time, especially when evaluating images across multiple registries. This guide covers all the practical ways to inspect remote images with Skopeo.

---

## Installing Skopeo

Ensure Skopeo is installed on your system before proceeding.

```bash
# Install on Fedora/RHEL

sudo dnf install -y skopeo

# Install on Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y skopeo

# Install on macOS
brew install skopeo

# Confirm installation
skopeo --version
```

## Basic Image Inspection

The simplest use of `skopeo inspect` retrieves high-level metadata about an image.

```bash
# Inspect an image on Docker Hub
skopeo inspect docker://docker.io/library/nginx:latest

# Inspect an image on Quay.io
skopeo inspect docker://quay.io/prometheus/prometheus:latest

# Inspect an image on a private registry
skopeo inspect docker://registry.example.com/myapp:v1.0
```

The output includes the image digest, creation date, architecture, OS, labels, environment variables, and layer information.

## Extracting Specific Fields with jq

The inspect output is JSON, so you can pipe it to `jq` for targeted extraction.

```bash
# Get just the image digest
skopeo inspect docker://docker.io/library/alpine:3.19 | jq -r '.Digest'

# Get the creation timestamp
skopeo inspect docker://docker.io/library/alpine:3.19 | jq -r '.Created'

# List all environment variables defined in the image
skopeo inspect docker://docker.io/library/postgres:16 | jq '.Env'

# Get the image labels (useful for version info, maintainers, etc.)
skopeo inspect docker://docker.io/library/nginx:latest | jq '.Labels'

# Get the architecture and OS
skopeo inspect docker://docker.io/library/node:20 | jq '{Arch: .Architecture, OS: .Os}'
```

## Inspecting Raw Manifests

The `--raw` flag returns the raw manifest from the registry, which is useful for understanding multi-architecture images and layer digests.

```bash
# Get the raw manifest (often a manifest list for popular images)
skopeo inspect --raw docker://docker.io/library/python:3.12 | jq .

# Check if an image is a manifest list or a single manifest
skopeo inspect --raw docker://docker.io/library/python:3.12 | \
  jq '.mediaType'

# List all platforms in a multi-arch manifest
skopeo inspect --raw docker://docker.io/library/python:3.12 | \
  jq '.manifests[] | {platform: .platform, digest: .digest}'
```

## Inspecting Image Configuration

The `--config` flag returns the full OCI image configuration, which includes the command, entrypoint, exposed ports, and build history.

```bash
# Get the full image configuration
skopeo inspect --config docker://docker.io/library/nginx:latest | jq .

# Extract the default command and entrypoint
skopeo inspect --config docker://docker.io/library/nginx:latest | \
  jq '{Cmd: .config.Cmd, Entrypoint: .config.Entrypoint}'

# List exposed ports
skopeo inspect --config docker://docker.io/library/redis:7 | \
  jq '.config.ExposedPorts'

# View the build history (shows each layer's creation command)
skopeo inspect --config docker://docker.io/library/alpine:3.19 | \
  jq '.history[] | .created_by'
```

## Inspecting Images in Private Registries

For private registries, Skopeo uses the same authentication as Podman.

```bash
# Log in to the private registry with Podman
podman login registry.example.com

# Inspect the private image (auth is read automatically)
skopeo inspect docker://registry.example.com/myapp:production

# Alternatively, pass credentials directly
skopeo inspect \
  --creds "myuser:mypassword" \
  docker://registry.example.com/myapp:production

# Use a specific auth file
skopeo inspect \
  --authfile /path/to/auth.json \
  docker://registry.example.com/myapp:production
```

## Comparing Images Across Registries

You can use Skopeo inspect to compare images in different registries to verify they are identical.

```bash
#!/bin/bash
# compare-images.sh - Compare image digests across two registries

IMAGE="myapp:v2.0"
REG1="registry-a.example.com"
REG2="registry-b.example.com"

# Get the digest from each registry
DIGEST1=$(skopeo inspect "docker://${REG1}/${IMAGE}" | jq -r '.Digest')
DIGEST2=$(skopeo inspect "docker://${REG2}/${IMAGE}" | jq -r '.Digest')

echo "Registry A digest: ${DIGEST1}"
echo "Registry B digest: ${DIGEST2}"

# Compare the two digests
if [ "$DIGEST1" = "$DIGEST2" ]; then
  echo "Images are identical."
else
  echo "Images differ - consider re-syncing."
fi
```

## Inspecting Insecure Registries

For development registries without proper TLS, use the TLS verification flag.

```bash
# Inspect an image on an insecure (HTTP) registry
skopeo inspect \
  --tls-verify=false \
  docker://localhost:5000/dev-image:latest

# Inspect using a custom certificate directory
skopeo inspect \
  --cert-dir=/etc/containers/certs.d/myregistry.local \
  docker://myregistry.local/myapp:latest
```

## Summary

Skopeo inspect is a powerful tool for examining container images without downloading them. It retrieves metadata, configuration, manifests, and layer information directly from any Docker-compatible registry. By combining it with `jq`, you can extract exactly the information you need. It integrates seamlessly with Podman authentication and supports both secure and insecure registries. Use it to evaluate images before pulling, compare images across registries, and audit image configurations in your container workflows.
