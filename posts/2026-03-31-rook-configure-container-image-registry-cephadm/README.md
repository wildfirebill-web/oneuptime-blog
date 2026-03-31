# How to Configure Container Image Registry for cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, cephadm, Container, Registry, Docker, Air-Gap

Description: Configure cephadm to pull Ceph container images from a custom or private registry, including air-gapped environments and authenticated registries.

---

## Why Configure a Custom Registry?

By default, cephadm pulls Ceph container images from `quay.io/ceph/ceph`. You need a custom registry when:

- Operating in an air-gapped environment with no internet access
- Using an internal mirror for corporate compliance
- Pulling from a private registry with authentication
- Testing with custom-built Ceph images

## Checking the Current Image

```bash
ceph config get mgr container_image
```

Or:

```bash
ceph orch ps | head -5
# Shows the image currently in use
```

## Setting a Custom Registry Image

```bash
ceph config set global container_image registry.example.com/ceph/ceph:v18
```

## Configuring Registry Authentication

For private registries requiring login:

```bash
ceph config set mgr mgr/cephadm/registry_url https://registry.example.com
ceph config set mgr mgr/cephadm/registry_username myuser
ceph config set mgr mgr/cephadm/registry_password mypassword
```

Or using the cephadm registry config command:

```bash
ceph cephadm registry-login \
  --registry-url https://registry.example.com \
  --registry-username myuser \
  --registry-password mypassword
```

## Air-Gapped Setup: Mirroring Images

Mirror the Ceph image to your internal registry:

```bash
# On an internet-connected machine
docker pull quay.io/ceph/ceph:v18.2.0

docker tag quay.io/ceph/ceph:v18.2.0 registry.internal.example.com/ceph/ceph:v18.2.0

docker push registry.internal.example.com/ceph/ceph:v18.2.0
```

Also mirror Prometheus and Grafana images used by the monitoring stack:

```bash
docker pull quay.io/prometheus/prometheus:latest
docker tag quay.io/prometheus/prometheus:latest registry.internal.example.com/prometheus/prometheus:latest
docker push registry.internal.example.com/prometheus/prometheus:latest
```

## Configuring Image Per Service Type

Override images for specific service types:

```yaml
service_type: prometheus
spec:
  image: registry.internal.example.com/prometheus/prometheus:v2.47.0
```

```yaml
service_type: grafana
spec:
  image: registry.internal.example.com/grafana/grafana:10.1.0
```

Apply:

```bash
ceph orch apply -i monitoring-images.yaml
```

## Verifying the Registry Config

```bash
ceph config dump | grep -E "registry|container_image"
```

Test that a node can pull the image:

```bash
cephadm pull --image registry.example.com/ceph/ceph:v18
```

## Summary

Configure cephadm to use a custom container registry by setting `container_image` in the Ceph config and providing registry credentials via `ceph cephadm registry-login`. For air-gapped deployments, mirror images from `quay.io/ceph/ceph` to your internal registry before bootstrapping or upgrading. Override images per service type via service specification YAML files.
