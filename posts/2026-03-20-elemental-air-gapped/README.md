# How to Configure Elemental for Air-Gapped Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Air-Gap, Kubernetes, Edge, Security

Description: Set up Elemental in air-gapped environments by mirroring required images and configuring offline registries.

## Introduction

Air-gapped deployments require all container images and OS artifacts to be available from internal registries. Elemental supports air-gapped operation through private registry mirrors, locally cached OS images, and pre-seeded boot media. This guide covers the complete setup for air-gapped Elemental deployments.

## Prerequisites

- Private container registry (Harbor, Nexus, or similar)
- Internal HTTP server for OS image distribution
- Access to mirror required images before going air-gapped

## Step 1: Mirror Elemental Images

```bash
# List all required Elemental images

ELEMENTAL_VERSION="1.4.0"
IMAGES=(
  "registry.suse.com/rancher/elemental-operator:${ELEMENTAL_VERSION}"
  "registry.suse.com/rancher/elemental-operator-webhook:${ELEMENTAL_VERSION}"
  "registry.suse.com/rancher/sle-micro:latest"
  "registry.suse.com/rancher/elemental-toolkit/elemental-cli:latest"
)

# Pull and retag for private registry
PRIVATE_REGISTRY="my-registry.internal.example.com"

for IMAGE in "${IMAGES[@]}"; do
  docker pull "${IMAGE}"
  
  # Extract image name without registry
  IMG_NAME=$(echo "${IMAGE}" | sed 's|registry.suse.com/||')
  
  docker tag "${IMAGE}" "${PRIVATE_REGISTRY}/${IMG_NAME}"
  docker push "${PRIVATE_REGISTRY}/${IMG_NAME}"
done
```

## Step 2: Configure Helm for Air-Gapped Install

```bash
# Download Elemental Helm chart
helm pull elemental-operator/elemental-operator \
  --untar \
  --untardir ./charts

# Modify values to use private registry
cat > airgap-values.yaml << 'EOF'
image:
  repository: my-registry.internal.example.com/rancher/elemental-operator
  tag: "1.4.0"

webhook:
  image:
    repository: my-registry.internal.example.com/rancher/elemental-operator-webhook
    tag: "1.4.0"

# Configure registry mirror
registryMirrors:
  "registry.suse.com":
    - "https://my-registry.internal.example.com"
EOF

# Install from local chart with air-gapped values
helm install elemental-operator ./charts/elemental-operator \
  --namespace elemental-system \
  --create-namespace \
  --values airgap-values.yaml
```

## Step 3: Configure Private Registry on Nodes

```yaml
# machine-registration-airgap.yaml
apiVersion: elemental.cattle.io/v1beta1
kind: MachineRegistration
metadata:
  name: airgap-nodes
  namespace: fleet-default
spec:
  config:
    cloud-config:
      write_files:
        # Configure registry mirrors for K3s/RKE2
        - path: /etc/rancher/k3s/registries.yaml
          content: |
            mirrors:
              docker.io:
                endpoint:
                  - "https://my-registry.internal.example.com"
              registry.k8s.io:
                endpoint:
                  - "https://my-registry.internal.example.com"
              registry.suse.com:
                endpoint:
                  - "https://my-registry.internal.example.com"
            configs:
              "my-registry.internal.example.com":
                auth:
                  username: robot$elemental
                  password: "your-registry-password"
                tls:
                  ca_file: /etc/ssl/certs/internal-ca.pem
          permissions: "0600"

    elemental:
      install:
        device: /dev/sda
        reboot: true
```

## Step 4: Build Air-Gapped Seed Images

```bash
# Configure Docker to use private registry
cat > /etc/docker/daemon.json << 'EOF'
{
  "registry-mirrors": ["https://my-registry.internal.example.com"],
  "insecure-registries": []
}
EOF

systemctl restart docker

# Build seed image from mirrored base
docker build \
  -t my-registry.internal.example.com/elemental-os:v1.0.0 \
  --build-arg BASE=my-registry.internal.example.com/rancher/sle-micro:latest \
  -f Dockerfile.elemental \
  .

# Build ISO from locally available image
docker run --privileged --rm \
  -v $(pwd):/workspace \
  my-registry.internal.example.com/rancher/elemental-toolkit/elemental-cli:latest \
  build-iso \
  --config /workspace/airgap-config.yaml \
  --output /workspace/ \
  my-registry.internal.example.com/elemental-os:v1.0.0
```

## Step 5: Internal HTTP Server for OS Distribution

```bash
# Set up Nginx to serve OS images internally
cat > /etc/nginx/conf.d/elemental.conf << 'EOF'
server {
    listen 80;
    server_name files.internal.example.com;

    location /elemental/ {
        root /var/www/html;
        autoindex on;
        # Support range requests for large files
        add_header Accept-Ranges bytes;
    }
}
EOF

# Copy OS images to serving directory
mkdir -p /var/www/html/elemental
cp elemental-seed.iso /var/www/html/elemental/
cp elemental-seed.raw /var/www/html/elemental/

systemctl restart nginx
```

## Conclusion

Air-gapped Elemental deployments require careful pre-staging of all required artifacts: Helm charts, container images, OS base images, and boot media. By mirroring all dependencies to an internal registry and configuring nodes to use it, you achieve a fully functional Elemental deployment with no external network access. This approach is essential for high-security environments, military deployments, and industrial sites with strict network isolation requirements.
